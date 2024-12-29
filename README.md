# master2.sh
master2.sh

#!/bin/bash

# Cleanup Kubernetes installation if any previous setup exists
echo "Cleaning up previous Kubernetes setup..."
sudo hostname master-node-1
sudo kubeadm reset -f
sudo apt-get purge kubeadm kubelet kubectl -y
sudo apt-get autoremove -y
sudo apt-get purge containerd -y
sudo apt-get autoremove -y

# Remove Kubernetes and containerd configuration files and directories
echo "Removing Kubernetes and containerd configuration files..."
sudo rm -rf /etc/cni
sudo rm -rf /opt/cni
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/kubelet
sudo rm -rf /etc/kubernetes

# Update and prepare the system
echo "Updating and upgrading system packages..."
sudo apt update && sudo apt upgrade -y

# Disable swap
echo "Disabling swap..."
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Install required packages for Kubernetes installation
echo "Installing necessary packages for Kubernetes..."
sudo apt install -y apt-transport-https ca-certificates curl

# Add Kubernetes repository and GPG key
echo "Adding Kubernetes repository..."
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Update package list and install Kubernetes components
echo "Installing Kubernetes components..."
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Install containerd
echo "Installing containerd..."
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd --no-pager

# Verify the installed versions
echo "Verifying installed versions..."
kubelet --version
kubeadm version
kubectl version --client

# Configure Kubernetes networking and IP forwarding
echo "Configuring Kubernetes networking..."
echo -e "br_netfilter" | sudo tee /etc/modules-load.d/k8s.conf && sudo modprobe br_netfilter
echo -e "net.bridge.bridge-nf-call-ip6tables = 1\nnet.bridge.bridge-nf-call-iptables = 1" | sudo tee /etc/sysctl.d/k8s.conf
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-kubernetes-ip-forward.conf
sudo sysctl --system

# Initialize the Kubernetes master node
echo "Initializing Kubernetes master node..."
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Configure kubectl for the master node
echo "Configuring kubectl..."
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Apply Calico networking
echo "Applying Calico networking..."
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Get the status of nodes
echo "Getting the status of nodes..."
kubectl get nodes

