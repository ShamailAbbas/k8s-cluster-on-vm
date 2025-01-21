# Kubernetes Cluster Setup on VMs Guide

This guide provides step-by-step instructions to set up a Kubernetes cluster using `kubeadm`. Follow along to configure your control plane, worker nodes, and install necessary components.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Edit Hosts File](#edit-hosts-file)
3. [Control Plane](#control-plane)
4. [Worker Node(s)](#worker-nodes)
5. [Installing a Container Runtime – Containerd](#installing-a-container-runtime-–-containerd)
   - [Installation Prerequisites](#installation-prerequisites)
   - [Step 1: Installing Containerd](#step-1-installing-containerd)
   - [Step 2: Installing runc](#step-2-installing-runc)
   - [Step 3: Installing CNI Plugins](#step-3-installing-cni-plugins)
   - [Step 4: Configuring the systemd cgroup driver](#step-4-configuring-the-systemd-cgroup-driver)
6. [Install kubeadm, kubelet, and kubectl](#install-kubeadm-kubelet-and-kubectl)
7. [Creating a Cluster with kubeadm](#creating-a-cluster-with-kubeadm)
8. [Install CNI – Calico](#install-cni-–-calico)
9. [Joining Your Other Nodes to the Cluster – Workers](#joining-your-other-nodes-to-the-cluster-–-workers)
10. [Test Your Cluster – Deploy nginx](#test-your-cluster-–-deploy-nginx)
11. [Extra Commands](#extra-commands)
12. [Update GitHub Containerd, CNI Network Plugins, and RunC](#update-github-containerd-cni-network-plugins-and-runc)

## Prerequisites

Ensure you meet the following requirements before proceeding:

- Linux host with minimum 2GB RAM and 2 (v)CPUs.
- Network connectivity between nodes.
- Unique hostname, MAC address, and `product_uuid` for each node.
- Disable swap by running:
  ```bash
  sudo swapoff -a
  sudo sed -i '/ swap / s/^/#/' /etc/fstab
  ```
- Open required ports as specified [here](https://kubernetes.io/docs/reference/networking/ports-and-protocols/).

## Edit Hosts File

Update your `/etc/hosts` file to include your node’s IP address and hostname:

```bash
sudo vi /etc/hosts
```

## Control Plane

Open the following ports for the control plane:

| Protocol | Direction | Port Range | Purpose                     | Used By                |
|----------|-----------|------------|-----------------------------|------------------------|
| TCP      | Inbound   | 6443       | Kubernetes API server       | All                    |
| TCP      | Inbound   | 2379-2380  | etcd server client API      | kube-apiserver, etcd   |
| TCP      | Inbound   | 10250      | Kubelet API                 | Self, Control plane    |
| TCP      | Inbound   | 10259      | kube-scheduler              | Self                   |
| TCP      | Inbound   | 10257      | kube-controller-manager     | Self                   |

## Worker Node(s)

Open the following ports for worker nodes:

| Protocol | Direction | Port Range   | Purpose          | Used By           |
|----------|-----------|--------------|------------------|-------------------|
| TCP      | Inbound   | 10250        | Kubelet API      | Self, Control plane |
| TCP      | Inbound   | 30000-32767  | NodePort Services| All               |

## Installing a Container Runtime – Containerd

### Installation Prerequisites

Load necessary modules:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Set sysctl params:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

### Step 1: Installing Containerd

Download and install Containerd:

```bash
cd /tmp
wget https://github.com/containerd/containerd/releases/download/v1.7.23/containerd-1.7.23-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.23-linux-amd64.tar.gz
sudo wget -O /usr/local/lib/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

### Step 2: Installing runc

Download and install runc:

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.2.0/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Step 3: Installing CNI Plugins

Download and install CNI plugins:

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.6.0/cni-plugins-linux-amd64-v1.6.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.6.0.tgz
```

### Step 4: Configuring the systemd cgroup driver

Create and edit the `config.toml` file:

```bash
sudo mkdir /etc/containerd && sudo touch /etc/containerd/config.toml
sudo containerd config default > /etc/containerd/config.toml
sudo vi /etc/containerd/config.toml
```

Set `SystemdCgroup = true` and restart containerd:

```bash
sudo systemctl restart containerd
```

## Install kubeadm, kubelet, and kubectl

Install the necessary Kubernetes components:

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

## Creating a Cluster with kubeadm

Initialize the control-plane node:

```bash
sudo kubeadm init --apiserver-advertise-address=192.168.0.10 --pod-network-cidr=192.168.0.0/16
```

## Install CNI – Calico

Install Calico CNI:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml
```

## Joining Your Other Nodes to the Cluster – Workers

Join worker nodes to the cluster:

```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

## Test Your Cluster – Deploy nginx

Deploy nginx to test the cluster:

```bash
kubectl apply -f https://k8s.io/examples/application/deployment.yaml
```

## Extra Commands

Commands for updating and managing components:

```bash
sudo kubeadm config images pull
kubectl get nodes
kubectl get pods --all-namespaces
```

## Update GitHub Containerd, CNI Network Plugins, and RunC

Check for the latest updates and apply them as necessary.

---

**Note:** A video tutorial covering these steps will be uploaded soon. Stay tuned!

**Documentation Source:** [Kubernetes Official Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

