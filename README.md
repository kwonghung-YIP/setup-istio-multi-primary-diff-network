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
- Install container runtime - Docker Engine

- Install kubeadm

- **[take snapshot]** Upto this point, this image is ready to clone to a worker node  
  the packages being installed after this snapshot is for control plane node only

- Install Istio

- Install k9s

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
  193.171.34.11 cluster1-ctrl-plane
  ...
  ```
- regenerated and get a unique machine-id

  ```bash
  sudo rm /etc/machine-id
  sudo systemd-machine-id-setup
  sudo systemd-machine-id-setup --print
  ```

- to verify 

```bash
ip addr show
ip route
sudo resolvectl dns
cat /etc/hosts
```

- **[take snapshot]**

## 4. Form the kubernetes cluster 1 & 2

## 5. Set up the primary-to-primary service mesh
