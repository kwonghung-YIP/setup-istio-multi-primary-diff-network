# Introduction

The [Istio/Install Multi-Primary on different network example](https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/) is about to form an Istio mesh on top of two kubernetes clusters. In this guide, we will go through how to set up these two clusters from scratch, and finally implement the example on it.  


## 1. Configuration

### 1.1 Component Version
Component | Version
-- | --
VMWare workstation | 16.1.2 build
Linux distribution | Ubuntu 20.04.2 LTS (focal)
Container runtime | Docker Engine 20.10.7
Kubernetes | v1.21.2
CNI | Weavenet v2.8.1
Load Balancer Implementation | MetalLB v0.10.2
Istio | v1.10.1

### 1.2 VMWare Network config (NAT - VMnet8):
Config | Value
-- | --
Network Address | 194.89.64.0/24
Default Gateway | 194.89.64.2/24
Boardcast Address | 194.89.64.255
DNS Server | 1.1.1.1, 8.8.8.8
DHCP Range | 194.89.64.128/24 - 194.89.64.254/24
Cluster1 MatelLB Ext IP Range | 194.89.64.81/24 - 194.89.64.100/24
Cluster2 MatelLB Ext IP Range | 194.89.64.101/24 - 194.89.64.120/24

### 1.3 Worker Nodes VM Settings:
Hostname | static IP | Core | Ram
-- | -- | -- | --
ubuntu-20042-base | 194.89.64.10/24 | - | -
cluster1-ctrl-plane | 194.89.64.11/24 | 2 | 4G
cluster1-worker-node01 | 194.89.64.12/24 | 2 | 4G
cluster2-ctrl-plane | 194.89.64.13/24 | 2 | 4G
cluster2-worker-node01 | 194.89.64.14/24 | 2 | 4G

## 2. Prepare the base image

### 2.1 Create an Ubuntu 20.04.2 LTS Virtual Machine
- Enable DHCP to get IP address
- Create an admin account, for my case is **hung**
- Install ssh server 

### 2.2 Apply the ssh public key for passwordless login 
[[ref]]()

1. Generate a ssh key with PuTTY Key Generator  
1. Save the private key with or without passphase protection  
1. Copy the public key into the file `~/.ssh/authorized_keys`  
1. Launch Pagent and add the private key just saved  
1. Save a new session and append the login before the hostname (e.g. hung@194.89.64.128)  

### 2.3 Stop sudo to prompt for password again 
[[ref]](https://askubuntu.com/questions/147241/execute-sudo-without-password)

1. Run `sudo visudo`  
1. Append `hung ALL=(ALL) NOPASSWD: ALL` at the end of the file  

### 2.4 Disable the swap 
[[ref]](https://serverfault.com/questions/684771/best-way-to-disable-swap-in-linux)

1. The step is necessary for initiate Kubernetes cluster
1. Run `sudo swapoff -a`  
1. Comment out swap setting in `/etc/fstab` to make the permanent change  
1. Run `free -h` to check the swap size

### 2.5 Switch the netplan config from dhcp client to static IP
[[ref]](https://www.linuxtechi.com/assign-static-ip-address-ubuntu-20-04-lts/)
  
1. Update the netplan config `/etc/netplan/00-installer-config.yaml`:
```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      addresses: [194.89.64.10/24] # <= the static IP assigned to this node
      gateway4: 194.89.64.2        # <= the default gateway
      nameservers:
        addresses: [1.1.1.1,8.8.8.8] # <= the nameserver entries here will be added as the DNS server in systemd-resolved
  version: 2
```
  
2. Run the following to apply the change without reboot
```bash
sudo netplan apply 
```

## 3 Install container runtime - Docker Engine  
[ref#1 - Container runtimes | Kubernetes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker)  
[ref#2 - Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)  
 
### 3.1 Install packages to allow apt download packages from HTTPS channel
```bash
sudo apt-get update
sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
```
  
### 3.2 Add Docker’s official GPG key
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
  
### 3.3 Add apt repository for Docker's stable release
```bash
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
  
### 3.4 Install docker engine
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
  
### 3.5 Verify docker engine by running the hello-world
```bash
sudo docker run hello-world
```
  
### 3.6 Update the docker daemon config, particular to use systemd as the cgroup driver
```bash
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```
  
### 3.7 Update systemd setting to auto start the docker service after reboot
```bash
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```
  
## 4. Install kubeadm [[ref]](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

### 4.1 Let iptables see bridged traffic
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
  
### 4.2 Install kubeadm, kubelet and kubectl
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
  
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
  
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
  
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 4.3 [take a VM snapshot as checkpoint] - snapshot#1
### 4.3 [take snapshot]** Upto this point, this image is ready to clone to a worker node  
  the packages being installed after this snapshot is for control plane node only

## 5. Install Istio
[[ref]](https://istio.io/latest/docs/setup/getting-started/)**  

```bash
curl -L https://istio.io/downloadIstio | sh -
  
sudo rm /usr/local/bin/istioctl
sudo ln -s `pwd`/istio-1.10.1/bin/istioctl /usr/local/bin/istioctl
```

## 6. Install k9s
```bash
curl -s https://api.github.com/repos/derailed/k9s/releases/latest | \
grep browser_download_url | \
grep Linux_x86_64 | \
cut -d : -f 2,3 | \
tr -d \" | \
wget -i - -O k9s.tar.gz

mkdir ~/k9s
tar -zxvf k9s.tar.gz -C ~/k9s
rm k9s.tar.gz

sudo rm /usr/local/bin/k9s
sudo ln -s `pwd`/k9s/k9s /usr/local/bin/k9s
```

### 6.2 [take a VM snapshot as checkpoint] - snapshot#2

## 7. Clone base image to the control plane and work node

### 7.1 Change the hostname
```bash
sudo hostnamectl set-hostname cluster1-ctrl-plane
```

### 7.2 Change the static IP in netplan config `/etc/netplan/00-installer-config.yaml`

### 7.3 Update the `/etc/hosts` to algin the hostname and static IP address
```bash
127.0.0.1 localhost
#127.0.1.1 cluster1-ctrl-plane
194.89.64.11 cluster1-ctrl-plane
...
```

### 7.4 Regenerated and get a unique machine-id
```bash
sudo rm /etc/machine-id
sudo systemd-machine-id-setup
sudo systemd-machine-id-setup --print
```

### 7.5 Verify the network setup: route table, systemd-resolved 

The node now should have the correct hostname, IP address, unique MAC & machine ID, and able to resolve the www.google.com domain and ping it.
```bash
ip link
ip addr show ens33
ip route
sudo resolvectl dns
cat /etc/hosts
ping www.google.com
```

### 7.6 [take a VM snapshot as checkpoint] - snapshot#2

## 8. Create 2 primary kubernetes cluster

### 8.1 Create cluster1 in cluster1-ctrl-plane  
```bash
sudo kubeadm config images pull
sudo kubeadm init
```

### 8.2 Join worker node cluster1-worker-node01 into cluster1  
```bash
# in case you need to print the kubectl join cluster command and token again 
sudo kubeadm token create --print-join-command
```

### 8.3 Install the CNI - weave net
[[ref]](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#install)  
  ```
  kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
  
  watch kubectl get pods -A
  ```

## 9. Install MetalLB  
  [MetalLB > Installation](https://metallb.universe.tf/installation/)  
  [MetalLB > Layer 2 Configuration](https://metallb.universe.tf/configuration/)  
  
### 9.1 Edit `kube-proxy`
```bash
kubectl edit configmap -n kube-system kube-proxy
```
  
### 9.2 Find and update the strictARP property in kube-proxy from false to true
```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```
  
### 9.3 Install MetalLB with manifest
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
  
watch kubectl get pods -A
```
  
### 9.4 Assign external IP range to MetalLB Load Balancer for cluster-1
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
      - 194.89.64.81-194.89.64.100
EOF
```  

### 9.5 Assign external IP range to MetalLB Load Balancer for cluster-2
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
      - 194.89.64.101-194.89.64.120
EOF
```

## 10. Verify the kubernetes DNS service  
[ref#1 - Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)  
[ref#2 - Troubleshooting Kubernetes Networking Issues](https://goteleport.com/blog/troubleshooting-kubernetes-networking/)  
  
```bash
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
kubectl exec -i -t dnsutils -- nslookup kubernetes.default
```
  
**[take snapshot]**
Repeat above steps for cluster#2

## 11. Merge cluster1 and cluster2 kubeconfig and place it into cluster1-ctrl-plane
- **[take snapshot]**  


## 12. Create common Root CA and 2 intermediate CA
[ref#1 Istio - Plug in CA Certificates](https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/)  

1. Create top Root CA cert and key
```bash
mkdir -p ~/istio-certs
sudo apt install make
cd istio-certs
make -f ~/istio-1.10.1/tools/certs/Makefile.selfsigned.mk root-ca
make -f ~/istio-1.10.1/tools/certs/Makefile.selfsigned.mk cluster1-cacerts
make -f ~/istio-1.10.1/tools/certs/Makefile.selfsigned.mk cluster2-cacerts
```

2. Create cluster1 intermediate CA cert and key
```bash
kubectl create namespace istio-system --context ${CTX_CLUSTER1}
kubectl create secret generic cacerts \
    --context ${CTX_CLUSTER1} \
    -n istio-system \
    --from-file=cluster1/ca-cert.pem \
    --from-file=cluster1/ca-key.pem \
    --from-file=cluster1/root-cert.pem \
    --from-file=cluster1/cert-chain.pem
```

3. Create cluster1 intermediate CA cert and key
```bash
kubectl create namespace istio-system --context ${CTX_CLUSTER2}
kubectl create secret generic cacerts \
    --context ${CTX_CLUSTER2} \
    -n istio-system \
    --from-file=cluster2/ca-cert.pem \
    --from-file=cluster2/ca-key.pem \
    --from-file=cluster2/root-cert.pem \
    --from-file=cluster2/cert-chain.pem
```

4. Compare the CA root cert of two cluster
```bash
diff \
  <(kubectl --context="${CTX_CLUSTER1}" -n istio-system get secret cacerts -ojsonpath='{.data.ca-cert\.pem}')\
  <(kubectl --context="${CTX_CLUSTER2}" -n istio-system get secret cacerts -ojsonpath='{.data.ca-cert\.pem}')
```

## 13. Install istio on multi-primary clusters running on different networks

_*Reference*_  
[Istio - install multi-primary on different network](https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/)  
[Istio - install multi-primary on the same network](https://istio.io/latest/docs/setup/install/multicluster/multi-primary/)  

1. Config cluster1 as primary
```bash
cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
EOF
  
istioctl install --context="${CTX_CLUSTER1}" -f cluster1.yaml
```
  
2. Install the east-west gateway in cluster1
```bash
samples/multicluster/gen-eastwest-gateway.sh \
  --mesh mesh1 --cluster cluster1 --network network1 | \
  istioctl --context="${CTX_CLUSTER1}" install -y -f -
```
  
3. Expose services in cluster1
```bash
kubectl --context="${CTX_CLUSTER1}" apply -n istio-system -f \
  samples/multicluster/expose-services.yaml
```

4. Config cluster2 as primary
```bash
cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster2
      network: network2
EOF
  
istioctl install --context="${CTX_CLUSTER1}" -f cluster2.yaml
```
  
5. Install the east-west gateway in cluster2
```bash
samples/multicluster/gen-eastwest-gateway.sh \
  --mesh mesh1 --cluster cluster2 --network network2 | \
  istioctl --context="${CTX_CLUSTER2}" install -y -f -
```
  
6. Expose services in cluster2
```bash
kubectl --context="${CTX_CLUSTER2}" apply -n istio-system -f \
  samples/multicluster/expose-services.yaml
```
  
7. Enable Endpoint Discovery
```bash
istioctl x create-remote-secret \
  --context="${CTX_CLUSTER1}" \
  --name=cluster1 | \
kubectl apply -f - --context="${CTX_CLUSTER2}"

istioctl x create-remote-secret \
  --context="${CTX_CLUSTER2}" \
  --name=cluster2 | \
kubectl apply -f - --context="${CTX_CLUSTER1}"
```

## 14. Verify the mesh service discovery and cross-cluster traffic
  [ref#3 Istio - verify installation](https://istio.io/latest/docs/setup/install/multicluster/verify/)  
  [ref#4 Istio - Triubleshooting Multicluster](https://istio.io/latest/docs/ops/diagnostic-tools/multicluster/)  

  Create the **sample** namespace, **helloworld** service and **sleep** deployment in both clusters
  ```bash
  kubectl create --context="${CTX_CLUSTER1}" namespace sample
  kubectl create --context="${CTX_CLUSTER2}" namespace sample

  kubectl label --context="${CTX_CLUSTER1}" namespace sample \
    istio-injection=enabled
  kubectl label --context="${CTX_CLUSTER2}" namespace sample \
    istio-injection=enabled

  kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample    
  kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
    
  kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/sleep/sleep.yaml -n sample
  kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/sleep/sleep.yaml -n sample
  ```
  
  Deploy Helloworld v1 into cluster1
  ```bash
  kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/helloworld/helloworld.yaml \
    -l version=v1 -n sample
  ```

  Deploy Helloworld v2 into cluster2
  ```bash
  kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/helloworld/helloworld.yaml \
    -l version=v2 -n sample
  ```
  
  Test the **hellowworld service** in cluster1 repeatly and you should find the return from both versions helloworld endpoints
  ```bash
  kubectl exec --context="${CTX_CLUSTER1}" -n sample -c sleep \
    "$(kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.sample:5000/hello
  ```

  And same case for cluster2
  ```bash
  kubectl exec --context="${CTX_CLUSTER2}" -n sample -c sleep \
    "$(kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.sample:5000/hello
  ```

-
