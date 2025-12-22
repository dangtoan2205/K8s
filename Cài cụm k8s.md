# Istall K8s:

> Cài ubuntu server
>- Lab K8s (Master) : 192.168.80.160
>- Lab K8s (Worker #1) : 192.168.80.159
>- Lab K8s (Postgree) : 192.168.81.152
>- Lab K8s (CI/CD) : 192.168.81.12

-----

> Luồng:
>- Dev push code
>- CI/CD build image
>- CI/CD deploy lên K8s
>- App kết nối PostgreSQL

## Chuẩn bị chung (tất cả node)

```
sudo apt update
sudo apt install -y containerd
sudo systemctl enable containerd --now
```

## Tắt swap:

```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## Cài kube:

```
sudo apt install -y ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Khởi tạo Kubernetes Master Node

### Chuẩn bị OS (Master & Worker)

```
# Tắt swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Bật IP forward (bắt buộc)
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

### Cài Container Runtime (containerd)

```
sudo apt update
sudo apt install -y containerd
sudo systemctl enable containerd --now
```

### Cấu hình containerd (khuyến nghị)

```
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

### Sửa file:

```
sudo nano /etc/containerd/config.toml
```

> Đảm bảo:

```
SystemdCgroup = true
sandbox_image = "registry.k8s.io/pause:3.10.1"
```

### Restart:

```
sudo systemctl restart containerd
```

### Cài Kubernetes components

```
sudo apt install -y ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Khởi tạo Master Node:
> *Chỉ chạy trên Master*

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

> *Chú ý: Lưu lại lệnh `kubeadm join` được in ra.*

### Cấu hình kubectl cho user:

```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Kiểm tra:

```
kubectl get nodes
```

### Cài Pod Network (Flannel):

---------------------- (Calico)

```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Chờ:

```
kubectl get pods -n kube-flannel
```

## Join Worker Nodes

> Thực hiện trên Worker 1

```
# Bật tạm thời
sudo sysctl -w net.ipv4.ip_forward=1

# Bật vĩnh viễn
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

### Kiểm tra:

```
cat /proc/sys/net/ipv4/ip_forward
```

> Phải là `1`.

### Load kernel module

```
sudo modprobe br_netfilter
```

### Bật vĩnh viễn:

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```

### Cấu hình sysctl cho bridge network

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s-network.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

### Restart containerd (trên tất cả node).

```
sudo systemctl restart containerd
```

### Dùng lệnh kubeadm join master đã in ra

```
sudo kubeadm join 192.168.80.160:6443 \
  --token 2tc1rn.xa1s8vk73209k2ay \
  --discovery-token-ca-cert-hash sha256:d19e5a14f0d117a144a130fce4389d6a904da1b74b616cf9c12dad331f42b3c1

```

### Kiểm tra:

```
kubectl get nodes
```

## Deploy app test (nginx)

### Tạo ConfigMap cho conf.d

> File `nginx-conf.yaml`

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
data:
  default.conf: |
    server {
      listen 80;
      server_name _;

      location / {
        return 200 "Nginx test OK\n";
      }
    }
```

Apply:

```
kubectl apply -f nginx-conf.yaml
```



## Cài PostgreSQL (ngoài K8s)

### Trên server PostgreSQL:


































































