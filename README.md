# Multi-Node Kubernetes Cluster Setup Using Kubeadm
This readme provides step-by-step instructions for setting up a multi-node Kubernetes cluster using Kubeadm.

## Overview
This guide provides detailed instructions for setting up a multi-node Kubernetes cluster using Kubeadm. The guide includes instructions for installing and configuring containerd and Kubernetes, disabling swap, initializing the cluster, installing calico, and joining worker nodes to the cluster.

## Prerequisites
Before starting the installation process, ensure that the following prerequisites are met:

- You have at least two Ubuntu 18.04 or higher servers available for creating the cluster.
- Each server has at least 2GB of RAM and 2 CPU cores.
- The servers have network connectivity to each other.
- You have root access to each server.

### Ports needs to be allowed in AWS Security Group
> Master Node
```
6443	Kubernetes API server
2379-2380	etcd server client API
10250	Kubelet API
10259	kube-scheduler
10257	kube-controller-manager
```
> Worker Node Ports:
```
10250	Kubelet API
10256	kube-proxy
30000-32767	NodePort Services
```
## Installation Steps: 
The following are the step-by-step instructions for setting up a multi-node Kubernetes cluster using Kubeadm and 
this installation guide is for Kubernetes v1.30.
Update the system's package list and install necessary dependencies using the following commands:

Switch to the root user:
```
sudo -s
sudo apt-get update -y
```
The default behavior of a kubelet was to fail to start if swap memory was detected on a node.You MUST disable swap if the kubelet is not properly configured to use swap

```
swapoff -a
```

To make this change persistent across reboots, make sure swap is disabled in config files like /etc/fstab
```
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
Disable SELINUX:

```
apt install selinux-utils
setenforce 0
```
We also need to install a container runtime into each node in the cluster so that Pods can run there.
Install Docker Engine:

```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
To install the latest version, run:
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
docker --version
usermod -aG docker ubuntu
systemctl status docker
systemctl enable docker
```

Kubernetes 1.30 requires that you use a runtime that conforms with the Container Runtime Interface (CRI).

```
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.17/cri-dockerd_0.3.17.3-0.ubuntu-bionic_amd64.deb
apt install ./cri-dockerd_0.3.17.3-0.ubuntu-bionic_amd64.deb
systemctl status cri-docker
systemctl enable cri-docker
```

1. Installing kubeadm, kubelet and kubectl
2. You will install these packages on all of your machines:
3. kubeadm: the command to bootstrap the cluster.
4. kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
5. kubectl: the command line util to talk to your cluster.

**NOTE:**

1. These instructions are for Kubernetes v1.30.
2. Update the apt package index and install packages needed to use the Kubernetes apt repository:

```	
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
#Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
#Add the appropriate Kubernetes apt repository. Please note that this repository have packages only for Kubernetes 1.30
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
#Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
#(Optional) Enable the kubelet service before running kubeadm:
sudo systemctl enable --now kubelet
```

### Creating a cluster with kubeadm(only on Master node)
```
kubeadm init --pod-network-cidr=192.168.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
```

1. To make kubectl work for our non-root user, run these commands, which are also part of the kubeadm init output:
2. Our Kubernetes control-plane has initialized successfully!
3. To start using your cluster, you need to run the following as a regular user:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
#Alternatively, if you are the root user, you can run:
export KUBECONFIG=/etc/kubernetes/admin.conf
```

1. We must deploy a Container Network Interface (CNI) based Pod network add-on so that your Pods can communicate with each other. Cluster DNS (CoreDNS) will not start up before a network is installed:
2. Calico is a networking and network policy provider:

Install the Tigera operator and custom resource definitions.
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/tigera-operator.yaml
```

Install Calico by creating the necessary custom resource
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/custom-resources.yaml
```

Confirm that all of the pods are running with the following command.
```
watch kubectl get pods -n calico-system
```

**Verify Installation**

Verify that all the pods are up and running:

```
kubectl get pods --all-namespaces
```

**Join Nodes:**

To add nodes to the cluster, run the kubeadm join command with the appropriate arguments on each node. The command will output a token that can be used to join the node to the cluster.

**To print token:**
```
kubeadm token create --print-join-command
```
