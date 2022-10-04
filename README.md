# Introduction

The [Istio/Install Multi-Primary on different network example](https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/) is about to form an Istio mesh on top of two kubernetes clusters. In this guide, we will go through how to set up these two clusters from scratch, and finally implement the example on it.  


## 1. Configuration

#### 1.1 Component Version
Component | Version
-- | --
VMWare workstation | 16.1.2 build
Linux distribution | Ubuntu 20.04.2 LTS (focal)
Container runtime | Docker Engine 20.10.7
Kubernetes | v1.21.2
CNI | Weavenet v2.8.1
Load Balancer Implementation | MetalLB v0.10.2
Istio | v1.10.1

#### 1.2 VMWare Network config (NAT - VMnet8):
Config | Value
-- | --
Network Address | 194.89.64.0/24
Default Gateway | 194.89.64.2/24
Boardcast Address | 194.89.64.255
DNS Server | 1.1.1.1, 8.8.8.8
DHCP Range | 194.89.64.128/24 - 194.89.64.254/24
Cluster1 MatelLB Ext IP Range | 194.89.64.81/24 - 194.89.64.100/24
Cluster2 MatelLB Ext IP Range | 194.89.64.101/24 - 194.89.64.120/24

#### 1.3 Worker Nodes VM Settings:
Hostname | static IP | Core | Ram | Disk
-- | -- | -- | -- | --
ubuntu-20042-base | 194.89.64.10/24 | - | - | -
cluster1-ctrl-plane | 194.89.64.11/24 | 2 | 4G | 20G
cluster1-worker-node01 | 194.89.64.12/24 | 2 | 4G | 20G
cluster2-ctrl-plane | 194.89.64.13/24 | 2 | 4G | 20G
cluster2-worker-node01 | 194.89.64.14/24 | 2 | 4G | 20G

## 2. Prepare the base image

#### 2.1 Create an Ubuntu 20.04.2 LTS Virtual Machine
- Enable DHCP to get IP address
- Create an admin account, for my case is **hung**
- Install ssh server 

#### [take a VM snapshot as checkpoint]

#### 2.2 Apply the ssh public key for passwordless login 

1. Generate a ssh key with PuTTY Key Generator  
1. Save the private key with or without passphase protection  
1. Copy the public key into the file `~/.ssh/authorized_keys`  
1. Launch Pagent and add the private key just saved  
1. Save a new session and append the login before the hostname (e.g. hung@194.89.64.128)  

#### 2.3 Stop sudo to prompt for password again 
_*References:*_  
[Ask Ubuntu - Execute sudo without password](https://askubuntu.com/questions/147241/execute-sudo-without-password)

1. Run `sudo visudo`  
1. Append `hung ALL=(ALL) NOPASSWD: ALL` at the end of the file  

#### 2.4 Disable the swap 
_*References:*_  
[ServerFlaut - Best way to disable swap in linux](https://serverfault.com/questions/684771/best-way-to-disable-swap-in-linux)

1. The step is necessary for initiate Kubernetes cluster
1. Run `sudo swapoff -a`  
1. Comment out swap setting in `/etc/fstab` to make the permanent change  
1. Run `free -h` to check the swap size

#### 2.5 Switch the netplan config from dhcp client to static IP
_*References:*_  
[How to Assign Static IP Address on Ubuntu 20.04 LTS](https://www.linuxtechi.com/assign-static-ip-address-ubuntu-20-04-lts/)
  
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

#### [take a VM snapshot as checkpoint]

## 3 Install container runtime - Docker Engine  
_*References:*_  
[Kubernetes - Container runtimes: Docker](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker)  
[Docker Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)  
 
#### 3.1 Install packages to allow apt download packages from HTTPS channel
```bash
sudo apt-get update
sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
```
  
#### 3.2 Add Docker’s official GPG key
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
  
#### 3.3 Add apt repository for Docker's stable release
```bash
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
  
#### 3.4 Install docker engine
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
  
#### 3.5 Verify docker engine by running the hello-world
```bash
sudo docker run hello-world
```
  
#### 3.6 Update the docker daemon config, particular to use systemd as the cgroup driver
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

#### 3.7 Grant current user into docker group
```bash
sudo usermod -aG docker $USER
```
  
#### 3.8 Update systemd setting to auto start the docker service after reboot
```bash
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```
  
## 4. Install kubeadm
_*References:*_  
[Kubernetes - Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

#### 4.1 Let iptables see bridged traffic
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
  
#### 4.2 Install kubeadm, kubelet and kubectl
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

#### [take a VM snapshot as checkpoint] - snapshot#1
Upto this point, we had installed all necessary components and this snapshot is ready to clone for worker node  

## 5. Install Istio
_*References:*_  
[Istio - Getting Started](https://istio.io/latest/docs/setup/getting-started/)  

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

#### [take a VM snapshot as checkpoint] - snapshot#2
On top of the worker node snapshot, we installed the istio and k9s and this snapshot is ready to clone to control plane

## 7. Clone snapshot to the control plane and work node

- Clone from snapshot#2 to nodes: cluster1-ctrl-plane, cluster2-ctrl-plane
- Clone from snapshot#1 to nodes: cluster1-worker-node01, cluster2-worker-node02

The following example takes cluster1-ctrl-plane node as example
#### 7.1 Change the hostname
```bash
sudo hostnamectl set-hostname cluster1-ctrl-plane
```

#### 7.2 Change the static IP in netplan config `/etc/netplan/00-installer-config.yaml`
```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      addresses: [194.89.64.11/24] # <= the static IP assigned to this node
      gateway4: 194.89.64.2        # <= the default gateway
      nameservers:
        addresses: [1.1.1.1,8.8.8.8] # <= the nameserver entries here will be added as the DNS server in systemd-resolved
  version: 2
```

#### 7.3 Update the `/etc/hosts` to algin the hostname and static IP address
```bash
127.0.0.1 localhost
#127.0.1.1 cluster1-ctrl-plane
194.89.64.11 cluster1-ctrl-plane
...
```

#### 7.4 Regenerated and get a unique machine-id
```bash
sudo rm /etc/machine-id
sudo systemd-machine-id-setup
sudo systemd-machine-id-setup --print
```

#### 7.5 Verify the network setup: route table, systemd-resolved 

The node now should have the correct hostname, IP address, unique MAC & machine ID, and able to resolve the www.google.com domain and ping it.
```bash
ip link
ip addr show ens33
ip route
sudo resolvectl dns
cat /etc/hosts
ping www.google.com
```

#### [take a snapshot of all 4 nodes as checkpoint]

## 8. Create Kubernetes cluster: cluster1 and cluster2
_*References:*_  
[Kubernetes - Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

#### 8.1 Create cluster1 in cluster1-ctrl-plane  
```bash
sudo kubeadm config images pull
sudo kubeadm init
```

#### 8.2 Join worker node cluster1-worker-node01 into cluster1  
```bash
# in case you need to print the kubectl join cluster command and token again 
sudo kubeadm token create --print-join-command
```

#### 8.3 Install the CNI - weave net
_*References:*_ 
[Weaveworks - Integrating Kubernetes via the Addon](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#install)  
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
  
watch kubectl get pods -A
```

## 9. Install MetalLB  
_*References:*_  
[MetalLB - Installation](https://metallb.universe.tf/installation/)  
[MetalLB - Layer 2 Configuration](https://metallb.universe.tf/configuration/)  
  
#### 9.1 Edit the `kube-proxy`
```bash
kubectl edit configmap -n kube-system kube-proxy
```
  
#### 9.2 Find and update the strictARP property in kube-proxy from false to true
```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```
  
#### 9.3 Install MetalLB with manifest
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
  
watch kubectl get pods -A
```
  
#### 9.4 Assign external IP range to MetalLB Load Balancer for cluster1
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

#### 9.5 Assign external IP range to MetalLB Load Balancer for cluster2
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
_*References:*_  
[Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)  
[Troubleshooting Kubernetes Networking Issues](https://goteleport.com/blog/troubleshooting-kubernetes-networking/)  

Test the DNS service in both cluster1 and cluster2
```bash
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
kubectl exec -i -t dnsutils -- nslookup kubernetes.default
```  
#### [take a snapshot of all 4 nodes as checkpoint]

## 11. Merge cluster1 and cluster2 kubeconfig and place it into cluster1-ctrl-plane
_*References:*_  
[Kubernetes - Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)

```yaml
apiVersion: v1
kind: Config
preferences: {}
current-context: admin@cluster1

contexts:
- context:
    cluster: cluster1
    user: cluster1-admin
  name: admin@cluster1
- context:
    cluster: cluster2
    user: cluster2-admin
  name: admin@cluster2

clusters:
- cluster:
    certificate-authority-data:  LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1EWXlNVEUyTVRNeU1Gb1hEVE14TURZeE9URTJNVE15TUZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTjNvClErLzFqd21wWHBWd0tNZUx5ME51cHBsYlBvRjZYMW12azg0SUlDVGlGQS9DS3k5eGdrVWZkV2E4cDFjK3Y1NFMKa1hMWE1xN01tN3hwZXFxZ2xXRE4zbncxNmlwTDBGSzk3Yk8zeVJKQTlNTWR1aTQzZnhlZzBjS1Vpc3F3L29JSQppNHRoTTRTOGNMT3NiaGhwNXlIVnVaczEwQkhDT21yQlhDTElLb01icHFlNjZtMTI2ZGZ3a0FYVWlSTmg0dFhwCkNNT0RKZDBLeE5Mc0RGRnR6SmZGWkVmUGJWald4SVRCb3VUWnRYV3QwSWlFa0JRb051LzZDaGlrUUEwSE1PS1QKVktCTVJQa3BZdTFCaTJCRmkrQlFyT1hkVmVtTHN3T3BuYlFZQXQrTzNBMnNtRVNzbzk4U1Q5aDR6VlRwd2tqNApMU0tGK3M0WjJLNlppWnR3bjdNQ0F3RUFBYU5DTUVBd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZDb0UxTitTd2pURndxZk13dWVwUFRJRkNiRlhNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFCTnJoWC94WDFxNzJDY2cvYnhzajVjTjZMVk1ZTEEvRXB0aE5mLzdEa2JHdlh4OW1rTApvOW5scUJkNXMxZExNaU1aRWNBWkVLY3ZYd25uNC9rYWZuZ01UdzlxdWl5SWo3Z2VzcUFTUlJmUm9CbG0vUk5TCmd3eG1ZQ1pXYnlQRmtENElKYmNmUjFBK2FYSWdDSlNVZkJFSWUvU1JlUk5UQStnWlpqeDlJRjNZdk1iRU00RXkKZjdIU3NjbGtkbkxNbG05anJXSFR1U3N5cXR4NlVMd1ZoK0FjYmYzMnY1S1F2KzhYbTF1SHU5elAwUmFiWGpQOQpibGxDRHNMdlhnVFNkWExsajNaRmd0alZiK0FMMHEySDZUSGdVZjNhV0pUS2NJZjFZL0VZQWpCcEVxdnVxSGp3CnhmK2xFL05IYmt1b2t0a2ZYTFBzK3NIazY5TU5KVDRyN1d1MAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://194.89.64.11:6443
  name: cluster1
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1EWXlNVEUzTkRFME9Gb1hEVE14TURZeE9URTNOREUwT0Zvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTFFmCnpTZzVaMjFRR2FlMXF3TUlyVmZDRVlMTWpFNmkrZDZzM3Z1eGNhS205M2NnMnFoUkplNnNhVXJIdVkwSnJZMm4KYmRkQU1LbXYvZW5vVCtjZzM4L2dXdnVrQ0J1U1pVRG9wNVlnR3RTdWk3ZDRFVHJySE8wZG03Mk1NLzBWelQrNwptTEpTekd3YlpLWWVIR254eGJOU2UzbTFvQVRoUjhLMXNaNUQ5ZXMzdW14UHIvUmJEejkrTVc5VzFjSXJyNWs5CklwSE53T3Qya1pIWi8xTy9lRGVaZlZzYTU0SUVNSXRRK1VDbHAzVEo3V1NmbXAvcGl0R2hMeGNGZGhweE1kcHkKSkMrMVNZaGFlU1dVMnJJYVBFL3NaSUpWb3lxR0lWU0ZpdEJlTHlKNVdreWtwZktPbm81V05PRnJ1ZXl2b0U4agpMWEEvYk5odk9zc2xPbnRtZU9jQ0F3RUFBYU5DTUVBd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZFL0FBL3RXUjBWbVprZkVIMzVsNTNDWUJRaVhNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFCdmZLbnlnNEFxNXFrRHphT2t4WmxLbFc2ZTlvMUplTGEzUzY5SlV1LzYzVDlwR3pacwpXb2d1Q3BwUjk0eFB6clRzRkxPSlFwWkVvdXBoNFJCK241Snp2MTNub1BHbG1tWDF0dlRJZEUxcVkrQ2I4MVpuCm0rTnRLNXplVVdOZlFIL2NpaVQzTW9vOGY3ZDJyYk5yUFoyZU14b1o4MkVlV2JrYzRxa0VmQzdlak9ieXJDalYKUkl6SnQ1QWIwSzZGTWJuL3FibkJ0ZFY2S2owYStHK1NjRXhMaFpEdVdmRlVMc052K2x2TVRlaGJWVmQrSmlKeQp3cDVDRUkreUM1dDE5WmNzQldhMEFRbVE2cTdsK0s3MFZRd0l1TWlRaWpTUGIvUWRtNy9RUlRpVW9PRmRDYlRZCkV4UVZ0RU5DZkdiL3k2aHp2N0JXY0dHb0FrM1FsN0NZSDAvZgotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://194.89.64.13:6443
  name: cluster2

users:
- name: cluster1-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJVENDQWdtZ0F3SUJBZ0lJQmJOYTJKdmpKSUF3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TVRBMk1qRXhOakV6TWpCYUZ3MHlNakEyTWpFeE5qRXpNak5hTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQTZYZ1dqWEZlTE13NGZJZGsKZkV1TVVKTzRmcFFaUmFqd000RVFqcXJwT05iVGRiUmltYURGYVBuRnY3OHZCejI4S2hNa3pYemtNd3NTMVM2aQo1OGdzd3pmdk5rd2FYTmVaQ3FoSEVNOGVVaUNoVC9oTExPdmdhRmFleGlLVEU3c0NlVFdGMXVnS0R1RHRhdXl0CkVqYlJzVTl3TEhJeCtiSjNJN2lZcjNZTHBFaDZ5eEtCeVFXSGJrYmRGOGJHMVJSbHhySndZMEc4Qms0SGtKaysKcFp3b0ZGdk03TDIyNHlCaU50RkNwSWg0UXN0Vk4rditGQk12K1BDdHNzcE5BM0ZyVUFFd1hXbGdyMVJyR1Q3ZQpURHJYYUZJa01sWWhUUzkwYlUxb0s4eXJEM21NZmdNOWZ1Yi9CWnE4MlV1U0g5ZmNQUHMwRXZZR0FZWjZ1MW9DCndQcHpBd0lEQVFBQm8xWXdWREFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RBWURWUjBUQVFIL0JBSXdBREFmQmdOVkhTTUVHREFXZ0JRcUJOVGZrc0kweGNLbnpNTG5xVDB5QlFteApWekFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBZDc5dXhXbHpNSkFJREo2em8vckw1QnRKRTVEU1JoSzVWaDBwCkJvRVJ0cHo0KzZWS3N0dFdncGpoRnN5dkZ4M1AzR1RqVUNoWTdvOXNrL0xTSCtiTDM3MUh4WFdQUllHYk5meVAKeDd3SGNWZFB5Slo2dVBxVHpUTEExWGhoSE02MDlEYWlEMDI3L1Y1M25OK3NOSHIvbm1sSHE2bWtvdHVFMDZoWQprMFZTNExYOW9lK0tjeEozMGJTNmVqWnlFMWtJNDdsSVJHUkF6Y2NDUlN1RUxxMDZVNjhNdk1TY2RxUVVpRXlsCmVTUFBkeG80T1RQVXhiVldPcUVhOVFkOExMdTF3RVRSaG1RMzErNUh5ZXZXbDJjNy9Lbk0xUEljWUt3S1F4eWMKUmNJcEpIYkNTYTJIR0w3Q1U2RldlcXVCdThnakVFNmg1NXRRTnVRR05BOWJucFk2b3c9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBNlhnV2pYRmVMTXc0Zklka2ZFdU1VSk80ZnBRWlJhandNNEVRanFycE9OYlRkYlJpCm1hREZhUG5Gdjc4dkJ6MjhLaE1relh6a013c1MxUzZpNThnc3d6ZnZOa3dhWE5lWkNxaEhFTThlVWlDaFQvaEwKTE92Z2FGYWV4aUtURTdzQ2VUV0YxdWdLRHVEdGF1eXRFamJSc1U5d0xISXgrYkozSTdpWXIzWUxwRWg2eXhLQgp5UVdIYmtiZEY4YkcxUlJseHJKd1kwRzhCazRIa0prK3Bad29GRnZNN0wyMjR5QmlOdEZDcEloNFFzdFZOK3YrCkZCTXYrUEN0c3NwTkEzRnJVQUV3WFdsZ3IxUnJHVDdlVERyWGFGSWtNbFloVFM5MGJVMW9LOHlyRDNtTWZnTTkKZnViL0JacTgyVXVTSDlmY1BQczBFdllHQVlaNnUxb0N3UHB6QXdJREFRQUJBb0lCQVFEajlVTmYrOCtPUWlEdApSbTJSQjFzTDJoQ01WeUtONTdRUk5mWHF0MnBjK3pVaGVtM0R2enpCa1EvS2QydjkwQU9IdVlWM3RuaENkbzkrCjQ3aGdSQTJnMTE2VVQ1NTJCSFVEK09iYXZNRElROS85NjF2TGtzeGNWQ2RYSXE4azFyWkZqME1OWVNkZys3SVYKY3Q1U0tJQjZkaXY2MmMxK0Z3bEpNWmF6eTdqMlA0a3MwWCtmR1BLMGJlQXl5M1VGWlBzdldmeDB4SkRKRFlkWQpvaHpJNXAvNStuaW9tMEdmOXNFNWZ4MkM1U1JnbFhobVNocldONHpqMUlodzhnTURjd04wTnpyRzVBaVoxeGp4CkdsM1VSSnJ4WmwxNGhydHBvU0xQNXgxT2lYUmJtRldqdm5oVnlSdVkzU2dkT1VkRjc4Tkg5dzZuUlZCRTU4bHkKeDI4a0xhWHhBb0dCQVB4dXIzcStMR2U1bytyaHd3UmN4NkNhUFZydUFhTmpaTnBtczIyYW1wUS9EdHJNZ0FGMgpnUHFEMUpGYjF6UnNoVmszYW10K0RBajR5Q3lJWHkzYUFpTGFpNS9YZWlRLytUUlFFWmtFelVhejNIaVpPQ0RjCk0wclo2Sy9wYU1ueXY4WTlXMjh2UTJyT2F2VThScXpJdkFmZHJPd0dvaWhLRFkrdHRJdFkyZEJ0QW9HQkFPekUKeXQyd2hRNDFtU3lSeGo4c2kwWU9kTHVIYUx5QzhkRHd5NExEUU5rTXN3NGlUa01wWE5hQjZmOWdTUmQwaGRCOQovTnYvQ09oL1VQOXZmYzRJQlhpTXB2Rzg1bDFlWHZDQWh6dWlqVy80eHU5K1A0a0RZOExGZzNqRFIzekJudVN1ClZoQ1hxUlFMTFQ5WHNOTWg3NjZMSUpWMWZQaG9jWnFHeGhSdzJvc3ZBb0dBZVZoNzRuVW93M1BwNkM4K29BbzUKckdwNHRBMVZuRVZiWmVHWXYwZGlwNERva3lWYkkxamtCNGozMWloZiswTnZsc09jMUs5eStaMGVITW94ZHNrbAozYnRSQXpXQjhZc1BNS2FNenhJUDI3ejZicjY0ekpNTjFSMkxUWVRXYXIzV2ttVk1YdFpKZ2o1WURDczlqakd3CnNkZE9HT2ZYYTZhdGZqUHlaa24vNnNFQ2dZRUExSVF1c3AxbVVFSzdvYzJXYTgzSGxMSVZCTjJkbk5iTHhnYmMKSkJxdGNpUjc4d3ZIdzNDMDY3VGdHMkNKT294VUw3ZGw1dkViUmRSQkY0VXpIbU1FeGhjNUlYRzBNOG9vM1NZQQpPLzdEaE9WL2FpZWZUNVBEVDJlSmdqT0ZUdTFiZVZjaDJQTEh5RDNmOXlMMmpBdkIzcUR5TmpTbVh6RWdCdHRCCm44ZEw0ZkVDZ1lBZXpTS01PWlIwM2kxNWY3VTFibGFNbEMxWjdmbTVDVFpXYmxHS2U0WjdJK0V6S3llLzBWM08KQXc3cEtzbW43UkZmTk5ZMytkM1hRQ2N6UTJvV1BBWURhNldYRkdKRU9CQ3oySjJqbGxxcXA1NkdmaHI1VGpLNAp5TGdQL0VmWllvR3BZTUw1YWZFN1liS1lzcHFwdVI1a09zOVhDdVo1T0tUSWdDaTBJand5MWc9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
- name: cluster2-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJVENDQWdtZ0F3SUJBZ0lJWGpwd2NrbjdiTVl3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TVRBMk1qRXhOelF4TkRoYUZ3MHlNakEyTWpFeE56UXhOVEJhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQTRRalE3ME05OTJXWXY5WEQKQ2hyWlpkTUNJZkdnRldjb1RMWk1SM09oc1luTkhYZ0IreUlRZXZFRjIxcXdMbkdHS0syZmowS0U1eWtIcHhBeQptM1QyUUlNS2dweGtBamJwNE1ENWFYNjBzaW1QRmtFRFJBeTNGRytZVEwrdU45WUE3QzhhaldPakNpUkhNTHgyCkdOeXNMOFJvdHNUWnhaZVdpNTlMTmhPaXdZT2RBNkdRQWJ5UEhITUgxUzQ0UXdmN2RodHBBdWZmWWRkc1pRS00KSEdjQXh2NndnN2Jjd1JvakZTY3duTTdDYkpUeUFOOENDMWd5M0xuU044RDRWRy8wNEZNVWEyOTU2U3dlSnloRgpKb2JqVWd1MVRVVGxBZFBjRUYza3MxSE5PVWRuK1d6STFySE5CYnQ2TENVbHpWL0pkMFJoM1dseU9WSHNSZWduClFDd3JCUUlEQVFBQm8xWXdWREFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RBWURWUjBUQVFIL0JBSXdBREFmQmdOVkhTTUVHREFXZ0JSUHdBUDdWa2RGWm1aSHhCOStaZWR3bUFVSQpsekFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBUXE0YnoxYnFoaUFjK3liM0NpdHp1YWhYK2dGbXlNY1NoQlBNCkZ4RktYY1o5dEJlMmNLc0czdlFxRnBZVDl0UlcwZ2Z2THZ4K1JDVExYMGZOTW5ISTdGQnE3QmxRVkZjWVpodmwKVGlqOEJUSjlPTFZyc2l4dm5zWG12NVkxR1pVaGpNc3hNWHBBRVpVTW56OVlwczREMkpmZThPU29iRzFUazIwQgo4Wm14emE0eld3bXhHMnB0cWhyV1RteUxyc0REcFhJV1huaWNIWmNCZWpaak1ITHhBQk9NVUFFblc4MjdxZHRwCnc0WmRJTVRrcGlyQmlNMmVNZlg5K09mSTR1U2l5MDh6YXFuNzh0YjRYN3ltWmFLMlROTGx6ZVI4em55VmFIRGUKQytwN1Rudm9oSlZRdUxKbitYOHpjUGkvYlEvMXdpUWhIUllBU0tiVy9JdkdWaEFNTkE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBNFFqUTcwTTk5MldZdjlYRENoclpaZE1DSWZHZ0ZXY29UTFpNUjNPaHNZbk5IWGdCCit5SVFldkVGMjFxd0xuR0dLSzJmajBLRTV5a0hweEF5bTNUMlFJTUtncHhrQWpicDRNRDVhWDYwc2ltUEZrRUQKUkF5M0ZHK1lUTCt1TjlZQTdDOGFqV09qQ2lSSE1MeDJHTnlzTDhSb3RzVFp4WmVXaTU5TE5oT2l3WU9kQTZHUQpBYnlQSEhNSDFTNDRRd2Y3ZGh0cEF1ZmZZZGRzWlFLTUhHY0F4djZ3ZzdiY3dSb2pGU2N3bk03Q2JKVHlBTjhDCkMxZ3kzTG5TTjhENFZHLzA0Rk1VYTI5NTZTd2VKeWhGSm9ialVndTFUVVRsQWRQY0VGM2tzMUhOT1VkbitXekkKMXJITkJidDZMQ1VselYvSmQwUmgzV2x5T1ZIc1JlZ25RQ3dyQlFJREFRQUJBb0lCQUhLL1ZockxCT3dFQ0ZHNQpwSXlnaUQ1ZHpIYVdpUFNnOTNHMmUwcnI4WVZnS1JGZndsTFdXZVQyeGUvR1hKUXlHeURlOTcvTFFZM0Y1RHNTCkRWd3IxZTJyWkU2WmhIMkVsdG1lVFErNEpsZTZ6VldoclJLa0VTOEFnSDZTTnpvTmk4YmpkZnltMDlvMkNYOFcKZW5uTy9KWVc1dlpiaGxnMUpmVG9NeWZOOTI0SXczWXJDcFdVNDl4KzZLdllSalNDeTIzY0E1a25pbTZLL0plVAovMW1QdTVsL05RWUwwTGlvellQYWRpWWs2djFaTUZRUEo2RUE1eHo3YjgxYzdEVmVuelNIK0xQOFJ0MEZQUHkzCnkxK1N0bkJLYmVVWFFJSk5IenBHM1VPWlZ3QXJVRGs1MHRPUURram1TUXJzQTM0dTFEeG5tdWMxS21SQ0N4TFUKZlRNMVFqa0NnWUVBL2tCVWJNT3NzMHJrSDlPT0wvQ3hKRndxWFVrdTZjbW1BVjlkQURzRXlkc1Njak5mMGFSbApwMUFFZ2hNZVhwU1ExOW9oREt5N1pEWlNZUlVCUDR1NEkwcENVNmZwZGpFbysybUdDN09SWS9BTzE1S04rK3NXClJoU2R5aFowRHVjNFBObWRwTFRndHJDTWFJd0ozYldZRkNJV1I1YlQ2UTJvbGJrRXZCT0tEZE1DZ1lFQTRwVUwKQ1dnRDNkU1NFWnpWOWx2NzNUV2RvR3hIOEZxakt4ME5naGYyNGFaN1F3SC9NNlhvc2tjSEhuL3IyNWRqN0RXTAowS0RmUk9vU3RkZzhwemxPYTZsWWZmNnJzcWtLOHd2ckh4WVk0NkxobDlxREpNU1RrRkczWDl4S2RROFN0R0c0ClY4eXJ0SGc3OEdKK0RhLzZNQ2NFMVR1UlYwWXh1ME96OERtWFpNY0NnWUF1TG9FblFHT2VMWHhDUzZzSUNqQWkKNnBySFZ3T3VjM0l6elo2VzdDRnlpTmhRNWdRQmtGcm1pU0pJZmpDRi9YWlJ2czFDQUI0SmxkUmd6ZS9zR3ZUWApkQ1dZREdmYmtCSmhtRWxBMXQwUnlnam9IemFyQzRpQU1qNTI5cDBlRitHZksrZjJndVJPU3NNMk9qbVFpK3VUCnZKMVBZNVlhUHVEZ1VUc0s3b0dsQVFLQmdHdGhmU2lKRGdRTVlPbE43YXppclB1S0ZGaloyRUlWZ216RlNRaVYKZU9BNStRS3BxSnQramtnbkZ6MmlIRkltYmltY3V0VTEySG9kZ0o2RGkwTXBDbnhGZG5YSHd2Rlo0YUdMelhNZgpFczZXKzlqdXF1WTY3MEFmS2d1WktBUlFEMnBEUVkwQ3A0RlExZjgzZmt2WVVYYU9sMkRDNlQ5Mk9jMW82WmI0CmhFSXpBb0dCQUxQRDNxc2l5ZVhQYUlSNHp4eWc1WFNOS3kwdFkyVzBDcnM0MlVxQ3JYODVLb3U4aWc3dGI3VjgKWHJzS1ZlZlFuaGdRNTExd3FwZEE1Tk1ld0xqZ3RZbFUweGdwR1lGakgwLy9BL0NrSThrRkxwSXo2UVVsLzNEKwpZVHh3T0h5M3JRc1NLNEFUaHFiZXhZU1BzU3huT3ZuRUNMcW1XSG9kMkg0VzlRcHcxS2hQCi0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
```
#### [take a snapshot of cluster1-ctrl-plane as checkpoint]

## 12. Create common Root CA and intermediate CA for cluster1/2
_*References:*_  
[Istio - Plug in CA Certificates](https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/)  

#### 12.1 Create top Root CA cert and key
```bash
mkdir -p ~/istio-certs
sudo apt install make
cd istio-certs
make -f ~/istio-1.10.1/tools/certs/Makefile.selfsigned.mk root-ca
make -f ~/istio-1.10.1/tools/certs/Makefile.selfsigned.mk cluster1-cacerts
make -f ~/istio-1.10.1/tools/certs/Makefile.selfsigned.mk cluster2-cacerts
```

#### 12.2 Create cluster1 intermediate CA cert and key
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

#### 12.3 Create cluster2 intermediate CA cert and key
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

#### 12.4 Compare the CA root cert of two cluster
```bash
diff \
  <(kubectl --context="${CTX_CLUSTER1}" -n istio-system get secret cacerts -ojsonpath='{.data.ca-cert\.pem}')\
  <(kubectl --context="${CTX_CLUSTER2}" -n istio-system get secret cacerts -ojsonpath='{.data.ca-cert\.pem}')
```

## 13. Install istio on multi-primary clusters running on different networks
_*References:*_  
[Istio - install multi-primary on different network](https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/)  
[Istio - install multi-primary on the same network](https://istio.io/latest/docs/setup/install/multicluster/multi-primary/)  

#### 13.1 Config cluster1 as primary
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
  
#### 13.2 Install the east-west gateway in cluster1
```bash
samples/multicluster/gen-eastwest-gateway.sh \
  --mesh mesh1 --cluster cluster1 --network network1 | \
  istioctl --context="${CTX_CLUSTER1}" install -y -f -
```
  
#### 13.3 Expose services in cluster1
```bash
kubectl --context="${CTX_CLUSTER1}" apply -n istio-system -f \
  samples/multicluster/expose-services.yaml
```

#### 13.4 Config cluster2 as primary
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
  
istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml
```
  
#### 13.5 Install the east-west gateway in cluster2
```bash
samples/multicluster/gen-eastwest-gateway.sh \
  --mesh mesh1 --cluster cluster2 --network network2 | \
  istioctl --context="${CTX_CLUSTER2}" install -y -f -
```
  
#### 13.6 Expose services in cluster2
```bash
kubectl --context="${CTX_CLUSTER2}" apply -n istio-system -f \
  samples/multicluster/expose-services.yaml
```
  
#### 13.7 Enable Endpoint Discovery
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
_*References:*_  
[Istio - verify installation](https://istio.io/latest/docs/setup/install/multicluster/verify/)  
[Istio - Triubleshooting Multicluster](https://istio.io/latest/docs/ops/diagnostic-tools/multicluster/)  

#### 14.1 Create the *sample* namespace, *helloworld* service and *sleep* deployment in both clusters
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
  
#### 14.2 Deploy Helloworld v1 into cluster1
```bash
kubectl apply --context="${CTX_CLUSTER1}" \
  -f samples/helloworld/helloworld.yaml \
  -l version=v1 -n sample
```

#### 14.3 Deploy Helloworld v2 into cluster2
```bash
kubectl apply --context="${CTX_CLUSTER2}" \
  -f samples/helloworld/helloworld.yaml \
  -l version=v2 -n sample
```
  
#### 14.4 Test the *hellowworld service* in cluster1 

While you test it repeatly, you should get return from both v1 running on cluster1 and v2 running on cluster2
```bash
kubectl exec --context="${CTX_CLUSTER1}" -n sample -c sleep \
  "$(kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l \
  app=sleep -o jsonpath='{.items[0].metadata.name}')" \
  -- curl -sS helloworld.sample:5000/hello
```

#### 14.5 And do the same for cluster2
```bash
kubectl exec --context="${CTX_CLUSTER2}" -n sample -c sleep \
  "$(kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l \
  app=sleep -o jsonpath='{.items[0].metadata.name}')" \
  -- curl -sS helloworld.sample:5000/hello
```

## 15. Check the istio-proxy sidecar config
```bash
kubectl config get-contexts
istioctl --context admin@cluster1 ps
kubectl --context admin@cluster1 get pod --namespace sample --output wide
kubectl --context admin@cluster2 get service --namespace istio-system
istioctl --context admin@cluster1 pc ep sleep-557747455f-4c7vz.sample --cluster="outbound|5000||helloworld.sample.svc.cluster.local"

kubectl config get-contexts
istioctl --context admin@cluster2 ps
kubectl --context admin@cluster2 get pod --namespace sample --output wide
kubectl --context admin@cluster1 get service --namespace istio-system
istioctl --context admin@cluster2 pc ep sleep-557747455f-jznfb.sample --cluster="outbound|5000||helloworld.sample.svc.cluster.local"
```


