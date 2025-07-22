# 🚀 Kubernetes Starter Guide

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com/)

> A comprehensive guide to setting up a Kubernetes cluster with master and worker nodes using kubeadm.

---

## 📋 Table of Contents

- [🚀 Kubernetes Starter Guide](#-kubernetes-starter-guide)
  - [📋 Table of Contents](#-table-of-contents)
  - [📖 Overview](#-overview)
  - [✅ Prerequisites](#-prerequisites)
  - [🔧 Installation Guide](#-installation-guide)
    - [Phase 1: System Prerequisites (All Nodes)](#phase-1-system-prerequisites-all-nodes)
    - [Phase 2: Container Runtime Setup](#phase-2-container-runtime-setup)
    - [Phase 3: Kubernetes Components Installation](#phase-3-kubernetes-components-installation)
    - [Phase 4: Master Node Configuration](#phase-4-master-node-configuration)
    - [Phase 5: Worker Node Setup](#phase-5-worker-node-setup)
    - [Phase 6: Cluster Verification](#phase-6-cluster-verification)
  - [🔧 Quick Commands Reference](#-quick-commands-reference)
  - [🛠️ Troubleshooting](#️-troubleshooting)
  - [📚 Additional Resources](#-additional-resources)

---

## 📖 Overview

This guide provides a complete walkthrough for setting up a production-ready Kubernetes cluster using `kubeadm`. You'll learn how to configure both master (control-plane) and worker nodes step by step.

### What You'll Build

- **Control Plane Node**: Manages the cluster, runs API server, scheduler, and controller manager
- **Worker Nodes**: Run your application pods and workloads
- **Network Plugin**: Calico CNI for pod networking and network policies

---

## ✅ Prerequisites

Before starting, ensure you have:

| Requirement    | Description                                |
| -------------- | ------------------------------------------ |
| **OS**         | Ubuntu 20.04/22.04 LTS                     |
| **RAM**        | Minimum 2GB (4GB+ recommended)             |
| **CPU**        | 2+ cores                                   |
| **Network**    | Private network connectivity between nodes |
| **Privileges** | sudo access on all nodes                   |

> ⚠️ **Important**: Disable swap on all nodes as Kubernetes requires it to be off.

---

## 🔧 Installation Guide

### Phase 1: System Prerequisites (All Nodes)

> 🎯 **Run on**: Master + Worker nodes

#### 1.1 Disable Swap

```bash
# Disable swap temporarily
sudo swapoff -a

# Make it permanent by commenting out swap entries
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Verify swap is disabled
free -h
```

#### 1.2 Configure Kernel Modules

```bash
# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Verify modules are loaded
lsmod | grep br_netfilter
lsmod | grep overlay
```

#### 1.3 Configure Network Settings

```bash
# Set sysctl parameters for Kubernetes networking
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply changes without reboot
sudo sysctl --system

# Verify the configuration
sudo sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

---

### Phase 2: Container Runtime Setup

> 🎯 **Run on**: Master + Worker nodes

#### 2.1 Install containerd

```bash
# Update package index
sudo apt-get update

# Install dependencies
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Add Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index and install containerd
sudo apt-get update
sudo apt-get install -y containerd.io
```

#### 2.2 Configure containerd

```bash
# Generate default configuration
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroups
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd

# Verify containerd is running
sudo systemctl status containerd
```

---

### Phase 3: Kubernetes Components Installation

> 🎯 **Run on**: Master + Worker nodes

#### 3.1 Add Kubernetes Repository

```bash
# Install required packages
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes GPG key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null
```

#### 3.2 Install Kubernetes Components

```bash
# Update package index
sudo apt-get update

# Install kubelet, kubeadm, and kubectl
sudo apt-get install -y kubelet kubeadm kubectl

# Hold packages to prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet service
sudo systemctl enable kubelet
```

---

### Phase 4: Master Node Configuration

> 🎯 **Run on**: Master node only

#### 4.1 Initialize the Cluster

```bash
# Initialize the cluster
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# 💡 Save the join command that appears at the end!
# It will look like: kubeadm join <IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

#### 4.2 Configure kubectl

```bash
# Set up kubeconfig for the current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Test kubectl access
kubectl get nodes
```

#### 4.3 Install Network Plugin (Calico)

```bash
# Install Calico CNI
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Wait for pods to be ready (this may take a few minutes)
kubectl get pods -n kube-system

# Check node status
kubectl get nodes
```

#### 4.4 Generate Join Command (if needed)

```bash
# Generate a new join command for worker nodes
kubeadm token create --print-join-command
```

---

### Phase 5: Worker Node Setup

> 🎯 **Run on**: Worker nodes only

#### 5.1 Reset Node (if previously configured)

```bash
# Reset the node if it was previously part of a cluster
sudo kubeadm reset

# Clean up
sudo rm -rf /etc/cni/net.d
sudo ipvsadm --clear
```

#### 5.2 Join the Cluster

```bash
# Use the join command from the master node (example)
sudo kubeadm join <MASTER-IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --v=5

# Example:
# sudo kubeadm join 192.168.1.100:6443 \
#   --token abcdef.0123456789abcdef \
#   --discovery-token-ca-cert-hash sha256:1234567890abcdef... \
#   --v=5
```

---

### Phase 6: Cluster Verification

> 🎯 **Run on**: Master node

#### 6.1 Verify Cluster Status

```bash
# Check all nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system

# Check cluster info
kubectl cluster-info

# Check cluster components
kubectl get componentstatuses
```

#### 6.2 Deploy a Test Application

```bash
# Create a test deployment
kubectl create deployment nginx-test --image=nginx

# Expose the deployment
kubectl expose deployment nginx-test --port=80 --type=NodePort

# Check the service
kubectl get services nginx-test

# Scale the deployment
kubectl scale deployment nginx-test --replicas=3

# Verify pods are distributed across nodes
kubectl get pods -o wide
```

---

## 🔧 Quick Commands Reference

### Essential Commands

| Command                                    | Description                     |
| ------------------------------------------ | ------------------------------- |
| `kubectl get nodes`                        | View cluster nodes              |
| `kubectl get pods -A`                      | View all pods across namespaces |
| `kubectl get services`                     | View services                   |
| `kubectl describe node <node-name>`        | Get detailed node information   |
| `kubectl logs <pod-name>`                  | View pod logs                   |
| `kubectl exec -it <pod-name> -- /bin/bash` | Access pod shell                |

### Troubleshooting Commands

```bash
# Check kubelet logs
sudo journalctl -u kubelet -f

# Check cluster events
kubectl get events --sort-by=.metadata.creationTimestamp

# Reset a node completely
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
sudo ipvsadm --clear

# Regenerate join token
kubeadm token create --print-join-command
```

---

## 🛠️ Troubleshooting

### Common Issues and Solutions

<details>
<summary><strong>🔍 Node Status Shows "NotReady"</strong></summary>

**Cause**: Usually indicates network plugin issues.

**Solution**:

```bash
# Check if Calico pods are running
kubectl get pods -n kube-system | grep calico

# If pods are not running, reinstall Calico
kubectl delete -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

</details>

<details>
<summary><strong>🔍 "Connection Refused" Error</strong></summary>

**Cause**: API server not accessible or kubeconfig issues.

**Solution**:

```bash
# Check if API server is running
sudo systemctl status kubelet

# Verify kubeconfig
kubectl config view

# Reconfigure kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

</details>

<details>
<summary><strong>🔍 Worker Node Join Fails</strong></summary>

**Cause**: Token expired, network connectivity, or firewall issues.

**Solution**:

```bash
# Generate new token on master
kubeadm token create --print-join-command

# Check connectivity from worker to master
telnet <master-ip> 6443

# Reset worker node and try again
sudo kubeadm reset
```

</details>

### Port Requirements

| Port      | Protocol | Direction | Purpose                 |
| --------- | -------- | --------- | ----------------------- |
| 6443      | TCP      | Inbound   | Kubernetes API server   |
| 2379-2380 | TCP      | Inbound   | etcd server client API  |
| 10250     | TCP      | Inbound   | kubelet API             |
| 10259     | TCP      | Inbound   | kube-scheduler          |
| 10257     | TCP      | Inbound   | kube-controller-manager |

---

## 📚 Additional Resources

### Documentation

- [Official Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubeadm Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Calico Documentation](https://docs.projectcalico.org/getting-started/kubernetes/)

### Useful Tools

- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [K9s - Terminal UI for Kubernetes](https://k9scli.io/)
- [Helm - Package Manager](https://helm.sh/)

### Next Steps

1. 🔐 **Security**: Set up RBAC and Network Policies
2. 📊 **Monitoring**: Install Prometheus and Grafana
3. 📦 **Package Management**: Install Helm
4. 🔄 **CI/CD**: Set up deployment pipelines
5. 💾 **Storage**: Configure persistent volumes

---

<div align="center">
  <strong>🎉 Congratulations! Your Kubernetes cluster is ready to use! 🎉</strong>
  
  Made with ❤️ for the Kubernetes community
</div>

---

> **Note**: This guide is tested on Ubuntu 20.04/22.04. For other distributions, package installation commands may differ.
