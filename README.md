# On-Prem Kubernetes Cluster Setup using kubeadm

This repository provides a complete **step-by-step guide** to build a production-grade **on-premises Kubernetes cluster** from scratch using **kubeadm**, **containerd**, and **Ubuntu 22.04/24.04**.

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

## ⚙️ Installation Steps

### 1. System Setup

```bash
sudo hostnamectl set-hostname cp1   # Adjust per node
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

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
