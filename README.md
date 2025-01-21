Here is the provided content converted into a Markdown (`.md`) format:

```markdown
# Table of Contents
- [Prerequisites](#prerequisites)
- [Edit hosts file](#edit-hosts-file)
- [Control plane](#control-plane)
- [Worker node(s)](#worker-nodes)
- [Installing a container runtime – Containerd](#installing-a-container-runtime--containerd)
  - [Installation prerequisites](#installation-prerequisites)
  - [Verify that the br_netfilter, overlay modules are loaded](#verify-that-the-br_netfilter-overlay-modules-are-loaded)
  - [Installing Containerd](#installing-containerd)
    - [Step 1: Installing containerd](#step-1-installing-containerd)
    - [Step 2: Installing runc](#step-2-installing-runc)
    - [Step 3: Installing CNI plugins](#step-3-installing-cni-plugins)
    - [Step 4: Configuring the systemd cgroup driver](#step-4-configuring-the-systemd-cgroup-driver)
- [Install kubeadm, kubelet and kubectl](#install-kubeadm-kubelet-and-kubectl)
- [Creating a cluster with kubeadm](#creating-a-cluster-with-kubeadm)
- [Install CNI – Calico](#install-cni--calico)
- [Joining your other nodes to the cluster – Workers](#joining-your-other-nodes-to-the-cluster--workers)
- [Test your cluster – deploy nginx](#test-your-cluster--deploy-nginx)
- [Extra commands](#extra-commands)
- [Update Github Containerd, CNI Network plugins, and RunC](#update-github-containerd-cni-network-plugins-and-runc)
- [Note](#note)

---

## Prerequisites
- Linux host
- Minimum 2GB RAM and 2 (v)CPUs
- Public or Private Network connectivity between nodes
- Unique hostname, MAC address, and product_uuid for every node.
  - Verify MAC address using: `ip link` or `ifconfig -a`
  - Check product_uuid using: `sudo cat /sys/class/dmi/id/product_uuid`
- Swap is supported since kubelet v1.22. As of v1.28, it is only supported for cgroup v2:
  - Check cgroup by running: `grep cgroup /proc/filesystems`
  - To be safe, turn off swap by running:
    ```bash
    sudo swapoff -a
    sudo sed -i '/ swap / s/^/#/' /etc/fstab
    ```
- Open required ports as below:
  - [Kubernetes Ports and Protocols Documentation](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)

---

## Edit hosts file
Add your IP to your `/etc/hosts` file with your hostname. The screenshot below is an example on a Virtualbox VM. In a public cloud environment, you should use an FQDN domain as your hostname.

```bash
sudo vi /etc/hosts
```

Add your private IP or public IP depending on which one you want kubelet to identify as your default node IP. On Virtualbox, add the Host-only IP.

---

## Control plane
| Protocol | Direction | Port Range | Purpose | Used By |
|----------|-----------|------------|---------|---------|
| TCP      | Inbound   | 6443       | Kubernetes API server | All |
| TCP      | Inbound   | 2379-2380  | etcd server client API | kube-apiserver, etcd |
| TCP      | Inbound   | 10250      | Kubelet API | Self, Control plane |
| TCP      | Inbound   | 10259      | kube-scheduler | Self |
| TCP      | Inbound   | 10257      | kube-controller-manager | Self |

Although etcd ports are included in the control plane section, you can also host your own etcd cluster externally or on custom ports.

---

## Worker node(s)
| Protocol | Direction | Port Range | Purpose | Used By |
|----------|-----------|------------|---------|---------|
| TCP      | Inbound   | 10250      | Kubelet API | Self, Control plane |
| TCP      | Inbound   | 30000-32767| NodePort Services† | All |

---

## Installing a container runtime – Containerd
Documentation source: [Kubernetes Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

Container runtime is the application responsible for running containers in Kubernetes.

### Installation prerequisites
Forwarding IPv4 and letting iptables see bridged traffic:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

### Verify that the br_netfilter, overlay modules are loaded by running:
```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

Verify that the `net.bridge.bridge-nf-call-iptables`, `net.bridge.bridge-nf-call-ip6tables`, and `net.ipv4.ip_forward` system variables are set to 1 in your sysctl config by running:

```bash
sudo sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### Installing Containerd
Documentation source: [Containerd Getting Started](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

After installing Containerd, you will also install runc and CNI plugins.

#### Step 1: Installing containerd
Download the binary for your amd64 or arm64 system from [Containerd Releases](https://github.com/containerd/containerd/releases). Ensure to download the latest releases from the provided link and switch it in place of the one below.

```bash
cd /tmp

# You can replace the version with the latest from GitHub releases page
wget https://github.com/containerd/containerd/releases/download/v1.7.23/containerd-1.7.23-linux-amd64.tar.gz

# Extract the containerd download
sudo tar Cxzvf /usr/local containerd-1.7.23-linux-amd64.tar.gz

# For systemd system users, add the unit file for containerd using either of the following commands:
sudo wget -O /usr/local/lib/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

# Or if you get the error: No such file or directory, this will work (Debian / Ubuntu):
sudo wget -O /usr/lib/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

# Finally, reload service daemons and enable containerd
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

#### Step 2: Installing runc
Download the right version for your system: amd64 or arm64. Assets download page: [runc Releases](https://github.com/opencontainers/runc/releases)

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.2.0/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

#### Step 3: Installing CNI plugins
Download the `cni-plugins—.tgz` archive from [CNI Plugins Releases](https://github.com/containernetworking/plugins/releases), and extract it under `/opt/cni/bin`:

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.6.0/cni-plugins-linux-amd64-v1.6.0.tgz

# Extract it/Install it
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.6.0.tgz
```

#### Step 4: Configuring the systemd cgroup driver
Setting cgroup for containerd for each Virtual machine. Cgroups (control groups) constrain resources allocated to Linux processes.

Based on how you installed containerd (using the binary as illustrated above), the default config file will not be available; you have to create it manually as shown below:

Switch to root first before running the next set of commands:

```bash
# Switch to root
sudo su -

# Create containerd config folder then generate a config file
sudo mkdir /etc/containerd && sudo touch /etc/containerd/config.toml && sudo containerd config default > /etc/containerd/config.toml

# Edit the generated containerd service configuration file /etc/containerd/config.toml:
sudo vi /etc/containerd/config.toml
```

Once the file is open, find the section shown below and change to `SystemdCgroup = true`. Make the value true and save. Easiest way is to search for `SystemdCgroup` and edit it in the file.

Save `/etc/containerd/config.toml`

Apply the changes by restarting containerd:

```bash
sudo systemctl restart containerd
```

You can log out of root or continue with the root shell, it is up to you.

```bash
# Log out of root
exit
```

---

## Install kubeadm, kubelet and kubectl
- `kubeadm`: the tool used to bootstrap the Kubernetes cluster.
- `kubelet`: the component that runs on all the machines in your cluster and does things like starting pods and containers.
- `kubectl`: the command line utility used in talking to your cluster.

Update apt package index and install prerequisite apps:

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg

# Download the public signing key for the Kubernetes package repositories:
sudo mkdir -p -m 755 /etc/apt/keyrings

# The version below (v1.30) does not affect the Kubernetes version to be installed
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Ubuntu Debian K8S repository. This version affects the Kubernetes components that will be installed.
# Change it (v1.30) to the current Kubernetes version you'd like to install
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

# Install kubeadm, kubelet, and kubectl and hold their versions to prevent the system from updating them.
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Enable the kubelet service before running kubeadm:
sudo systemctl enable --now kubelet
```

You have added apt repository for packages intended for v1.30. If you want to use the latest Kubernetes, go to the Kubernetes documentation and get the latest repository. Link: [Kubernetes Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/). Or just change the version where indicated above.

---

## Creating a cluster with kubeadm
Source docs: [Kubernetes Create Cluster with Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

Side note: Before running `kubeadm init`, you can pull all the images needed to bootstrap a kubeadm cluster:

```bash
sudo kubeadm config images pull
```

Initialize kubeadm cluster: To initialize the control-plane node, run:

```bash
sudo kubeadm init --apiserver-advertise-address=192.168.0.10 --pod-network-cidr=192.168.0.0/16
```

Change the pod network CIDR and your control plane’s IP as needed. You may also leave out these options, and the default values will be used.

---

## Install CNI – Calico
Source documentation: [Calico Documentation](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)

```bash
# Install the operator on your cluster.
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml

# Download the custom resource needed to configure Calico.
curl https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml -O

# Apply the manifest to install Calico.
kubectl create -f custom-resources.yaml
```

If you wish to customize the Calico install, customize the downloaded `custom-resources.yaml` manifest locally.

Verify Calico installation in your cluster. Check/Confirm as the pods get created:

```bash
watch kubectl get pods -n calico-system
```

---

## Joining your other nodes to the cluster – Workers
1. SSH to the machine.
2. Become root (e.g., `sudo su -`).
3. Install a runtime if needed. We’ve already installed it up there.
4. Run the command that was output by `kubeadm init`. For example:

```bash
kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

If you do not have the token, you can get it by running the following command on the control-plane node:

```bash
kubeadm token create --print-join-command
```

Tokens expire after 24 hours. You can create a new token by running the following command on the control-plane node.

---

## Test your cluster – deploy nginx
In any folder, create a file called `mynginx.yaml`. In this file, copy and paste the following:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30000
```

In the directory with the above file, run:

```bash
kubectl apply -f mynginx.yaml
```

On your local browser, visit `machine_ip:30000`. You should be able to see the nginx welcome page.

To delete the deployment, run:

```bash
kubectl delete -f mynginx.yaml
```

---

## Extra commands
If you check the Kubelet status (`sudo systemctl status kubelet`) and it is not running, check journal logs for what might be wrong:

```bash
sudo journalctl -f -u kubelet
```

On Windows, disable Hyper-V:

```bash
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-Hypervisor
```

Source: [Microsoft Documentation](https://learn.microsoft.com/en-us/troubleshoot/windows-client/application-management/virtualization-apps-not-work-with-hyper-v)

---

## Update Github Containerd, CNI Network plugins, and RunC
Copy the following, update the different GitHub release versions to the current ones. Then run it on each node. Wherever there is `wget`, ensure you update to the latest version available on GitHub.

```bash
# Go to temp directory
cd /tmp

# Download the latest version of containerd
wget https://github.com/containerd/containerd/releases/download/v1.7.23/containerd-1.7.23-linux-amd64.tar.gz

# Extract it
sudo tar Cxzvf /usr/local containerd-1.7.23-linux-amd64.tar.gz

sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Updating/Installing runc
wget https://github.com/opencontainers/runc/releases/download/v1.2.0/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

# CNI - Download and extract
wget https://github.com/containernetworking/plugins/releases/download/v1.6.0/cni-plugins-linux-amd64-v1.6.0.tgz
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.6.0.tgz
```

Then reboot all your nodes.

---

## Note
I will upload a video with these steps. Check back soon.

Documentation source: [Kubernetes Kubeadm Installation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
```
