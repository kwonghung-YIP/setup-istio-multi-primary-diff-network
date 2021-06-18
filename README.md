# istio-installation-guide

This guide is about how to set up two k8s clusters, then deploy an istio mesh using the primary-to-primary model.

The version:

VMWare workstation: 
Ubuntu: 20.04 LTS
Container runtime: Docker Engine
Kubernetes:
CNI: Weavenet
Matel LB
Istio

## 1. Plan the network layout

## 2. Prepare the base image
- **Create a new Ubuntu 20.04 LTS Virtual Machine**

- **Copy the ssh public key [[ref]]()**  

- **Disable the sudo to ask for password again [[ref]](https://askubuntu.com/questions/147241/execute-sudo-without-password)**  
  `sudo visudo`

- **Disable the swap [[ref]](https://serverfault.com/questions/684771/best-way-to-disable-swap-in-linux)**  
  run `sudo swapoff -a`  
  comment out swap setting in `/etc/fstab` to make the permanent change  
  run `free -h` to check the swap size

- **Assign the VM with a static IP [[ref]](https://www.linuxtechi.com/assign-static-ip-address-ubuntu-20-04-lts/)**  
  update the netplan config `/etc/netplan/00-installer-config.yaml`:
  
  ```yaml
  # This is the network config written by 'subiquity'
  network:
    ethernets:
      ens33:
        addresses: [193.171.34.13/24] # <= the static IP assigned to this node
        gateway4: 193.171.34.2        # <= the default gateway
        nameservers:
          addresses: [1.1.1.1,8.8.8.8] # <= the nameserver entries here will be added as the DNS server in systemd-resolved

    version: 2
  ```
- **Install container runtime - Docker Engine**  
  [ref#1 - Container runtimes | Kubernetes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker)  
  [ref#2 - Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)  
 
  Install packages to allow apt download packages from HTTPS channel
  ```bash
  sudo apt-get update
  sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
  ```
  
  Add Docker’s official GPG key
  ```bash
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
  ```
  
  Add apt repository for Docker's stable release
  ```bash
  echo \
    "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  ```
  
  Install docker engine
  ```bash
  sudo apt-get update
  sudo apt-get install docker-ce docker-ce-cli containerd.io
  ```
  
  Verify docker engine by running the hello-world
  ```bash
  sudo docker run hello-world
  ```
  
- **Install kubeadm [[ref]](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)**

  Let iptables see bridged traffic
  ```bash
  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
  br_netfilter
  EOF

  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  sudo sysctl --system
  ```
  
  Install kubeadm, kubelet and kubectl
  ```bash
  sudo apt-get update
  sudo apt-get install -y apt-transport-https ca-certificates curl
  
  sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
  
  echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
  
  sudo apt-get update
  sudo apt-get install -y kubelet kubeadm kubectl
  sudo apt-mark hold kubelet kubeadm kubectl
  ```

- **[take snapshot]** Upto this point, this image is ready to clone to a worker node  
  the packages being installed after this snapshot is for control plane node only

- **Install Istio [[ref]](https://istio.io/latest/docs/setup/getting-started/)**  

  ```bash
  curl -L https://istio.io/downloadIstio | sh -
  ```

- **Install k9s**  

- **[take snapshot]**

## 3. Clone base image to the control plane and work node

- change the hostname

  ```bash
  sudo hostnamectl set-hostname cluster1-ctrl-plane
  ```

- change the static IP in netplan config `/etc/netplan/00-installer-config.yaml`

- update the /etc/hosts to algin the hostname and fixed IP address

  ```bash
  127.0.0.1 localhost
  #127.0.1.1 cluster1-ctrl-plane
  193.171.34.11 cluster1-ctrl-plane
  ...
  ```
- regenerated and get a unique machine-id

  ```bash
  sudo rm /etc/machine-id
  sudo systemd-machine-id-setup
  sudo systemd-machine-id-setup --print
  ```

- Verify the network setup: route table, systemd-resolved 

  ```bash
  ip link
  ip addr show ens33
  ip route
  sudo resolvectl dns
  cat /etc/hosts
  ping www.google.com
  ```

- **[take snapshot]**

## 4. Create 2 primary kubernetes cluster

- Create cluster#1 in cluster1-ctrl-plane  

  ```bash
  sudo kubeadm config images pull
  sudo kubeadm init
  ```

- join worker node cluster1-worker-node01 into cluster#1  

  ```bash
  # in case you need to print the kubectl join cluster command and token again 
  sudo kubeadm token create --print-join-command
  ```

- **Install the CNI - weave net [[ref]](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#install)**  
  ```
  kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
  
  watch kubectl get pods -A
  ```

- **Install MetalLB**  
  [MetalLB > Installation](https://metallb.universe.tf/installation/)  
  [MetalLB > Layer 2 Configuration](https://metallb.universe.tf/configuration/)  
  
  Enable strict ARP mode
  ```bash
  # edit the kube-proxy configmap
  kubectl edit configmap -n kube-system kube-proxy
  
  # update the strictARP property from false to true
  apiVersion: kubeproxy.config.k8s.io/v1alpha1
  kind: KubeProxyConfiguration
  mode: "ipvs"
  ipvs:
    strictARP: true
  ```
  
  Install MetalLB with manifest
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
  kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
  
  watch kubectl get pods -A
  ```
  
  Define external IP range assigned by MetalLB
  ```bash
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
    kind: ConfigMap
  metadata:
    namespace: metallb-system
    name: config
  data:
    config: |
      address-pools:
      - name: default
        protocol: layer2
        addresses:
        - 193.171.34.100-193.171.34.120
  EOF
  ```  

- **Verify the kubernetes DNS service**  
  [ref#1 - Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)  
  [ref#2 - Troubleshooting Kubernetes Networking Issues](https://goteleport.com/blog/troubleshooting-kubernetes-networking/)  
  
  ```bash
  kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
  kubectl exec -i -t dnsutils -- nslookup kubernetes.default
  ```
  
- **[take snapshot]**
- Repeat above steps for cluster#2

## 5. Set up the primary-to-primary service mesh [[ref]](https://istio.io/latest/docs/setup/install/multicluster/multi-primary/)

-
