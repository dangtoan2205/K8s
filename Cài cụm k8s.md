# 1. Thông tin lab:

> Hệ điều hành: Ubuntu Server
>- Lab K8s (Master) : `192.168.80.160`
>- Lab K8s (Worker #1) : `192.168.80.159`
>- Lab K8s (Postgree) : `192.168.81.152`
>- Lab K8s (CI/CD) : `192.168.81.12`

-----

# 2. Luồng hệ thống:
>- Dev push code
>- CI/CD build image
>- CI/CD deploy lên K8s
>- Application kết nối PostgreSQL

# 3. Chuẩn bị chung (tất cả node)

## 3.1. Cài container runtime

```
sudo apt update
sudo apt install -y containerd
sudo systemctl enable containerd --now
```

## 3.2. Tắt swap (bắt buộc):

```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## 3.3. Cấu hình kernel & sysctl (bắt buộc cho Calico)

```
# Load module
sudo modprobe br_netfilter

# Load vĩnh viễn
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

# Sysctl cho Kubernetes + Calico
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

> Kiểm tra:

```
cat /proc/sys/net/ipv4/ip_forward
# Phải là 1
```

# 4. Cài Kubernetes components (TẤT CẢ NODE):

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

# 5. Cấu hình containerd (TẤT CẢ NODE – khuyến nghị)

```
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

 > Sửa file:

```
sudo nano /etc/containerd/config.toml
```

> Đảm bảo:

```
SystemdCgroup = true
sandbox_image = "registry.k8s.io/pause:3.10.1"
```

> Restart:

```
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

# 6. Khởi tạo Kubernetes Master

> Chỉ chạy trên Master (192.168.80.160)

```
sudo kubeadm init
```
>⚠️ Không cần --pod-network-cidr khi dùng Calico (mặc định OK) </br>
📌 Lưu lại lệnh kubeadm join

## 6.1 Cấu hình kubectl cho user

```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> Kiểm tra:

```
kubectl get nodes
```

# 7. Cài Pod Network – CALICO (MASTER)

> ✅ Calico là network chính thức cho cluster này

```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

> Theo dõi:

```
kubectl get pods -n kube-system | grep calico
```

> Kết quả mong đợi:

```
calico-node                Running
calico-kube-controllers    Running
```

# 8. Join Worker Node

> Thực hiện trên Worker #1 (192.168.80.159)

```
sudo kubeadm join 192.168.80.160:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

## 8.1 Kiểm tra cluster (MASTER)

```
kubectl get nodes
```

> Kết quả:

```
k8s-master    Ready
k8s-worker1   Ready
```

# 9. Deploy app test (Nginx)

## 9.1. Tạo file nginx-conf.yaml

```
sudo nano nginx-conf.yaml
```

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

## 9.2. Tạo file nginx-deploy.yaml

```
sudo nano nginx-deploy.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          command: ["nginx", "-g", "daemon off;"]
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-conf
              mountPath: /etc/nginx/conf.d
      volumes:
        - name: nginx-conf
          configMap:
            name: nginx-conf
```

## 9.3. Tạo file nginx-svc.yaml

```
sudo nano nginx-svc.yaml
```

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-test-svc
spec:
  type: NodePort
  selector:
    app: nginx-test
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

## 9.4. Thứ tự nên apply

```
1. ConfigMap
2. Deployment
3. Service
```

```
kubectl apply -f nginx-conf.yaml
kubectl apply -f nginx-deploy.yaml
kubectl apply -f nginx-svc.yaml
```

## 9.5. Kiểm tra

### 9.5.1. Kiểm tra Pod

```
kubectl get pods
```

> Pod phải:

```
STATUS: Running
READY: 1/1
```

### 9.5.2. Kiểm tra Service

```
kubectl get svc nginx-test-svc
```

> Ghi chú:
> - NodePort có thể truy cập qua bất kỳ node nào
> - Trong lab, nên test từ node nội bộ trước

### 9.5.3. Test nội bộ (Trên Master)

```
kubectl exec -it <nginx-pod> -- curl http://localhost
```

> Kết quả:

```
Nginx test OK
```

### 9.5.4. Test NodePort từ node

```
curl http://<NODE_IP>:30080
```

> Ví dụ:

```
curl http://192.168.80.159:30080
```

# Cài PostgreSQL (ngoài K8s)

### Trên server PostgreSQL:


































































