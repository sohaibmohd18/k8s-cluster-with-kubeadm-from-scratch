# On-Prem Kubernetes Cluster Setup using kubeadm

This repository provides a complete **step-by-step guide** to build a production-grade **on-premises Kubernetes cluster** from scratch using **kubeadm**, **containerd**, and **Ubuntu 22.04/24.04**.

⚠️ Troubleshooting Tips at the end!!!

---

## Cluster Overview

| Role | Hostname | IP Address | Description |
|------|-----------|-------------|--------------|
| Control Plane | cp1 | 10.0.0.10 | Kubernetes API Server, etcd, Controller, Scheduler |
| Worker Node | w1 | 10.0.0.11 | Runs workloads |
| Worker Node | w2 | 10.0.0.12 | Runs workloads |
| Worker Node | w3 | 10.0.0.13 | Runs workloads |
| Worker Node | w4 | 10.0.0.14 | Runs workloads |

>  For HA setups, use 3 control-plane nodes and a virtual IP (VIP) via **keepalived** or **HAProxy**.

---

## Prerequisites

- 5 VMs with Ubuntu 22.04 or 24.04 (2 CPU, 4GB RAM min)
- Root or sudo privileges
- Stable internet connection
- Static IPs configured on all VMs
- Swap disabled on all nodes
- Ports open: `6443`, `2379–2380`, `10250`, `10257`, `10259`, `30000–32767`

---

## Installation Steps - For Control Plane

### 1. System Setup

```bash
sudo hostnamectl set-hostname cp1   # Adjust per node
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

```bash
# Kernel modules and sysctl
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

### 2. Install containerd

```bash
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl enable --now containerd
```

### 3. Install Kubernetes Components

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 4. Initialize Control Plane

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12
```

```bash
# Setup kubectl access
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 5. Deploy CNI (Networking)

```bash
# Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
OR

```bash
# Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

YOUR CONTROL PLANE IS READY - Now you need to initialize the Worker Nodes and join them with the Control Plane to form a Cluster

---

## Installation Steps for Worker Nodes

### 1. Prepare the System

```bash
# Set hostname
sudo hostnamectl set-hostname w1   # change to w2, w3, etc.
```

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

```bash
# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

```bash
# Sysctl settings for Kubernetes networking
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

### 2. Install and Configure Containerd

```bash
sudo apt-get update
sudo apt-get install -y containerd
```

```bash
# Configure containerd to use systemd cgroups
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

```bash
# Enable containerd
sudo systemctl enable --now containerd
```

### 3. Install Kubeadm, Kubelet and Kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Enable the kubelet service
```bash
sudo systemctl enable --now kubelet
```

REPEAT Step 1 - Step 3 IN ALL YOUR WORKER NODES

---

## Join Your Cluster

From your control-plane node, run this to get the join command:
```bash
kubeadm token create --print-join-command
```

You'll get something like:
```bash
kubeadm join 10.0.0.10:6443 --token abcd12.34efgh5678ijklmn \
    --discovery-token-ca-cert-hash sha256:1a2b3c4d5e6f7g8h9i0j...
```

Now run that entire command in each of your Worker Nodes

```bash
sudo kubeadm join 10.0.0.10:6443 \
  --token abcd12.34efgh5678ijklmn \
  --discovery-token-ca-cert-hash sha256:1a2b3c4d5e6f7g8h9i0j...
```

If joined, you'll see something like shown below

```bash
This node has joined the cluster!
```

### Verify from Control Plane

On your control plane

```bash
kubectl get nodes -o wide
```

You should see output like

```bash
NAME   STATUS   ROLES           AGE   VERSION
cp1    Ready    control-plane   25m   v1.30.0
w1     Ready    <none>          2m    v1.30.0
w2     Ready    <none>          1m    v1.30.0
```


YOUR CLUSTER IS READY NOW!!!

---

## Optional Addons

Metrics Server
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Ingress Controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### etcd Backup

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/backups/etcd-$(date +%F).db
```

---

## Troubleshooting Tips

**Problem**                                              **Fix**
The connection to the server localhost:8080 was refused    Ensure you’re running kubectl on control plane with proper kubeconfig (/etc/kubernetes/admin.conf)
Worker not showing up                                      Check worker logs: sudo journalctl -u kubelet -f
Token expired                                              Recreate token: kubeadm token create --print-join-command
Network not ready                                          Check CNI pods: kubectl get pods -n kube-flannel or -n calico-
