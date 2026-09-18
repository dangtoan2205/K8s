Tổng quan Kubernetes (K8s)
---

Kubernetes là nền tảng orchestration dùng để:
- Deploy container
- Scale hệ thống tự động
- Self-healing (tự phục hồi)
- Load balancing
- Quản lý networking, storage, secret, config

Kiến trúc K8s được chia thành 2 phần chính:
```
+----------------------+
|   Control Plane      |
| (Quản lý Cluster)    |
+----------------------+
          |
          |
+----------------------+
|      Worker Node     |
| (Chạy ứng dụng)      |
+----------------------+
```

# 1. KIẾN TRÚC TỔNG THỂ K8S

<img width="750" height="761" alt="image" src="https://github.com/user-attachments/assets/4e320896-57c1-4b34-9419-5c355ad517b2" />

<img width="770" height="717" alt="image" src="https://github.com/user-attachments/assets/e2fdfb93-c6fb-4415-af70-7f9cb4a6aecd" />

<img width="1290" height="860" alt="image" src="https://github.com/user-attachments/assets/e0bb7559-0b79-40ae-bf81-08da170f48a7" />

# 2. CONTROL PLANE (MASTER NODE)

Control Plane là “bộ não” của cluster.

Nhiệm vụ:
- Quản lý toàn bộ cluster
- Scheduling
- Theo dõi trạng thái
- API quản trị
- Điều phối container
- Thành phần chính

## 2.1 kube-apiserver

Là cổng giao tiếp trung tâm của K8s.

Mọi thao tác đều đi qua API Server:
```
kubectl apply
kubectl get pods
helm install
```
→ đều gọi tới:
```
kube-apiserver
```
Vai trò:
- Nhận request
- Validate
- Authentication / Authorization
- Giao tiếp etcd
- Trả kết quả

## 2.2 etcd

Database dạng key-value.

Lưu:
- Config cluster
- Secret
- Pod info
- Node info
- Deployment state

Ví dụ:
```
desired replicas = 3
```
Nếu etcd mất:
→ cluster gần như “mất não”.

Thực tế production:
- luôn backup etcd
- chạy HA etcd cluster

## 2.3 kube-scheduler

Scheduler quyết định:

Pod sẽ chạy ở node nào

Dựa vào:

CPU
RAM
Affinity
Taints/Tolerations
Resource requests
Node labels

Ví dụ:

frontend pod -> worker-01
backend pod -> worker-02
2.4 kube-controller-manager

Theo dõi trạng thái cluster.

Ví dụ:

Pod chết
Node down
Replica thiếu

→ controller sẽ tự tạo lại.

Ví dụ:

Deployment replicas = 3

Hiện chỉ còn:

2 pods

→ controller tạo thêm 1 pod mới.

Đây là:

Self-healing
2.5 cloud-controller-manager (nếu cloud)

Dùng khi chạy trên:

Amazon Web Services
Google Cloud
Microsoft Azure

Quản lý:

Load Balancer
Volume
Cloud networking
Node lifecycle
3. WORKER NODE

Worker node là nơi chạy workload thực tế.

Ví dụ:

Web app
API
Database
Monitoring
CI/CD
4. THÀNH PHẦN TRONG WORKER NODE

<img width="788" height="412" alt="image" src="https://github.com/user-attachments/assets/b2f11040-8789-4733-b1a4-7abf01c3250a" />

<img width="1024" height="648" alt="image" src="https://github.com/user-attachments/assets/064e9524-ad17-4c19-b16f-d77ea2260ca4" />

<img width="1290" height="860" alt="image" src="https://github.com/user-attachments/assets/15567801-6c2a-484d-ac74-fd58cd1bd9cf" />

4.1 kubelet

Agent chạy trên từng node.

Nhiệm vụ:

Nhận yêu cầu từ API Server
Tạo Pod
Monitor Pod
Báo trạng thái về Control Plane

Có thể hiểu:

kubelet = quản lý node local
4.2 kube-proxy

Xử lý networking.

Nhiệm vụ:

Service networking
Load balancing nội bộ
IPTables/IPVS rules

Ví dụ:

Service frontend

kube-proxy sẽ route traffic tới:

frontend-pod-1
frontend-pod-2
frontend-pod-3
4.3 Container Runtime

Runtime để chạy container.

Phổ biến:

containerd
CRI-O
Docker (cũ)

Hiện nay:

containerd = phổ biến nhất
5. POD — ĐƠN VỊ NHỎ NHẤT

Pod là đơn vị deploy nhỏ nhất trong K8s.

Một Pod có thể chứa:

1 container
hoặc nhiều container

Ví dụ:

Nginx container
Sidecar logging container
6. DEPLOYMENT

Deployment quản lý Pod.

Ví dụ:

replicas: 3

K8s sẽ luôn đảm bảo:

luôn có 3 pods

Deployment hỗ trợ:

Rolling update
Rollback
Scaling
7. SERVICE

Pod IP thay đổi liên tục.

Service tạo:

IP cố định

để app giao tiếp ổn định.

Các loại:

ClusterIP
NodePort
LoadBalancer
ExternalName
8. INGRESS

Ingress dùng để:

expose HTTP/HTTPS
reverse proxy
routing domain

Ví dụ:

api.company.com -> backend-service
admin.company.com -> admin-service

Thực tế enterprise:

NGINX Ingress
Traefik
HAProxy
Kong

Với background hiện tại của bạn đang dùng:

NGINX
Cloudflare
pfSense

thì flow thực tế thường sẽ là:

Cloudflare
   ↓
Firewall (pfSense/FortiGate)
   ↓
Ingress Controller
   ↓
Service
   ↓
Pods
9. NETWORKING TRONG K8S

Một trong những phần khó nhất của K8s.

Mô hình network
Pod ↔ Pod
Pod ↔ Service
External ↔ Ingress

Mỗi Pod có:

1 IP riêng

CNI phổ biến:

Calico
Flannel
Cilium

Production enterprise:

Calico hoặc Cilium
10. STORAGE

Container mặc định:

ephemeral

→ restart là mất data.

K8s dùng:

PV (Persistent Volume)
PVC (Persistent Volume Claim)

Storage backend:

NFS
Ceph
Longhorn
AWS EBS
11. KIẾN TRÚC HA (HIGH AVAILABILITY)

Production thường:

Control Plane
3 master nodes
Worker
N worker nodes
etcd
3 hoặc 5 nodes

Ví dụ enterprise:

3 Control Plane
5 Worker
HAProxy
Keepalived
12. FLOW THỰC TẾ KHI DEPLOY APP

<img width="1024" height="698" alt="image" src="https://github.com/user-attachments/assets/c9c252fe-e813-4ca7-9934-65c67b98f61a" />

<img width="1085" height="860" alt="image" src="https://github.com/user-attachments/assets/44bc10b6-9eeb-4c57-8bd7-6811dcdadf54" />

<img width="1024" height="1249" alt="image" src="https://github.com/user-attachments/assets/32182df4-b467-4032-a082-3a1ace11962c" />

Flow:

Dev push code
    ↓
CI/CD build image
    ↓
Push image lên registry
    ↓
kubectl apply deployment
    ↓
API Server nhận request
    ↓
etcd lưu desired state
    ↓
Scheduler chọn node
    ↓
kubelet tạo Pod
    ↓
container runtime chạy container
13. KIẾN TRÚC K8S THỰC TẾ ENTERPRISE

Với doanh nghiệp ~300 user như hạ tầng bạn đang quản lý, mô hình thường sẽ là:

Internet
   ↓
Cloudflare
   ↓
FortiGate/pfSense
   ↓
HAProxy / Nginx
   ↓
K8s Cluster
      ├── Frontend
      ├── Backend API
      ├── Redis
      ├── Monitoring
      ├── GitLab Runner
      ├── Jenkins
      └── Internal services

Monitoring:

Prometheus
Grafana
Loki
AlertManager
14. NHỮNG KHÓ NHẤT KHI TRIỂN KHAI K8S THỰC TẾ

Thực tế khó nhất không phải:

kubectl apply

mà là:

Networking
Ingress
DNS
Storage
HA
Security
Monitoring
CI/CD
Backup/Restore
Observability

Đặc biệt với background hiện tại của bạn:

VLAN
Reverse Proxy
pfSense
Nginx
Docker
CI/CD

thì bạn sẽ học K8s nhanh hơn khá nhiều vì đã hiểu:

network flow
reverse proxy
container
infrastructure thinking
15. LỘ TRÌNH HỌC K8S THỰC TẾ

Đề xuất thứ tự:

Docker fundamentals
Linux networking
YAML
Pod
Deployment
Service
Ingress
ConfigMap / Secret
Persistent Volume
Helm
Monitoring
CI/CD
HA cluster
Security
GitOps
16. TÓM TẮT NGẮN GỌN
Control Plane
   = quản lý cluster

Worker Node
   = chạy ứng dụng

Pod
   = đơn vị chạy container

Deployment
   = quản lý Pod

Service
   = networking nội bộ

Ingress
   = expose HTTP/HTTPS

etcd
   = database cluster

Scheduler
   = chọn node

Controller
   = self-healing

















