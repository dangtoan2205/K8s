# K8s


## 1. Kiểm tra mạng trong K8s (CNI):

### 1.1 Kiểm tra CNI có chạy không

```
kubectl get pods -n kube-system
```

> 👉 Nhìn pod:
> - calico (recommend)
> - flannel
> - cilium
> - cilium

Pod Running = mạng K8s OK.

### 1.2 Xem loại CNI đang dùng

```
kubectl get pods -n kube-system | grep kube-proxy
```

### 1.3 Xem IP pod

```
kubectl get pods -o wide
```

### 1.4 Kiểm tra Pod

```
kubectl get pods
```

### 1.5 Kiểm tra Service

```
kubectl get svc nginx-test-svc
```

## 2. Ingress

### 2.1 Khi nào KHÔNG cần

- nginx-test chỉ dùng để:
  - Test cluster
  - Test network (Calico)
  - Test Service / NodePort
- Không phải app thực tế
- Không cần domain / HTTPS

> Khi đó
>- NodePort là đủ
>- Cấu hình bạn đang có là CHUẨN

### 2.2 Trường hợp NÊN cài Ingress

- Muốn minh hoạ kiến trúc chuẩn
- Muốn:
  - Truy cập bằng domain
  - Dùng port 80/443
  - Nhiều app dùng chung entrypoint
- Muốn chuẩn bị cho production / CI-CD

> Khi đó
> - Nginx-test đóng vai trò demo Ingress
> - Ingress giúp:
> - Thống nhất kiến trúc
> - Làm ví dụ cho app thật sau này















