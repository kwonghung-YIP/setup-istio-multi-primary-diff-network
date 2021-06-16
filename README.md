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
- Create a new Ubuntu 20.04 LTS Virtual Machine

- Disable the swap [[ref]](https://serverfault.com/questions/684771/best-way-to-disable-swap-in-linux)  
  run `sudo swapoff -a`  
  comment out swap setting in `/etc/fstab`

- Assign the VM with a static IP [[ref]](https://www.linuxtechi.com/assign-static-ip-address-ubuntu-20-04-lts/)  
  update the `/etc/netplan/00-installer-config.yaml` as following
  
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
- Install container runtime - Docker Engine
- Install kubeadm

- Copy the ssh public key [[ref]]()  

- Disable the sudo to ask for password again [[ref]](https://askubuntu.com/questions/147241/execute-sudo-without-password)  
  `sudo visudo`

take snapshot

- Install Istio
- Install k9s

take snapshot

## 3. Clone base image to the control plane and work node

- change the hostname
  sudo hostnamectl set-hostname cluster1-ctrl-plane
- change the fixed IP in netplan configuration
- update the /etc/hosts to algin the hostname and fixed IP address
- remote and regenerated the /etc/machine-id

## 4. Form the kubernetes cluster 1 & 2

## 5. Set up the primary-to-primary service mesh
