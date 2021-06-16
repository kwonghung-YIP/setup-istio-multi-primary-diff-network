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
- Disable the sudo to ask for password again
  sudo visudo
- Assign the VM with a fixed IP
  update the /etc/netplan/00.conf as following
```bash
```
- Copy the ssh public key

- Install the Docker Engine
- Install kubeadm

take snapshot

- Install Istio
- Install k9s

## 3. Clone base image to the control plane and work node

- change the hostname
  sudo hostnamectl set-hostname cluster1-ctrl-plane
- change the fixed IP in netplan configuration
- update the /etc/hosts to algin the hostname and fixed IP address
- remote and regenerated the /etc/machine-id

## 4. Form the kubernetes cluster 1 & 2

## 5. Set up the primary-to-primary service mesh
