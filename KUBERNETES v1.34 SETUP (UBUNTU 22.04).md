📦 KUBERNETES v1.34 SETUP (UBUNTU 22.04)
-----

# 🔧 PHẦN 1 — ALL NODES (MASTER + WORKER)

> **Chạy trên TẤT CẢ các node**

```bash
# =========================
# 1. Disable swap (Kubernetes requirement)
# =========================
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# =========================
# 2. Load required kernel modules
# =========================
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# =========================
# 3. Set required sysctl params
# =========================
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sudo sysctl --system

# =========================
# 4. Install containerd (CRI)
# =========================
sudo apt update
sudo apt install -y containerd

# =========================
# 5. Configure containerd for Kubernetes
# =========================
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# IMPORTANT: Use systemd cgroup (required by kubeadm)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

# =========================
# 6. Enable & restart containerd
# =========================
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable containerd
sudo systemctl restart containerd

# Verify containerd
sudo systemctl status containerd --no-pager

# =========================
# 7. Install Kubernetes packages (v1.34)
# =========================
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet (it will start after kubeadm init/join)
sudo systemctl enable kubelet
```

**✅ Sau bước này:**
- containerd chạy OK
- kubelet sẵn sàng
- chưa có cluster (đúng mong đợi)

# ☸️ PHẦN 2 — MASTER NODE SETUP

> **Chỉ chạy trên node master**

```bash
# =========================
# 1. Initialize Kubernetes cluster
# =========================
sudo kubeadm init \
  --kubernetes-version=v1.34.0 \
  --pod-network-cidr=192.168.0.0/16

# =========================
# 2. Configure kubectl for current user
# =========================
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Test
kubectl get nodes
```

⛔ Lúc này node sẽ **NotReady** → vì **chưa cài CNI** (bình thường).

# 🌐 Cài Network Plugin (Calico)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Chờ 1–2 phút, kiểm tra:

```bash
kubectl get pods -A
kubectl get nodes
```

✅ Node master → `Ready`


# 🧑‍🔧 PHẦN 3 — WORKER NODE JOIN

> **Chạy trên các worker node**

Trên **master**, lấy join command:

```bash
kubeadm token create --print-join-command
```

Ví dụ output:

```bash
kubeadm join 10.10.0.3:6443 --token xxxx \
  --discovery-token-ca-cert-hash sha256:yyyy
```

**👉 Copy & chạy lệnh đó trên worker node.** 

# 🔍 PHẦN 4 - KIỂM TRA CUỐI (MASTER)

```bash
kubectl get nodes
kubectl get pods -A
```

**Kết quả mong đợi:**
- Tất cả node Ready
- kube-apiserver / etcd / controller / scheduler Running
- calico-node Running
