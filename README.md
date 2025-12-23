K8s
-
> Để chạy app cần:
>- Deployment.yaml => Ví dụ: 04-deployment.yaml
>- Service.yaml => Ví dụ: 05-service.yaml

> Nếu có config:
>- ConfigMap.yaml => Ví dụ: 01-configmap.yaml

> Nếu có mật khẩu/token:
>- Secret.yaml => Ví dụ: 02-secret.yaml

> Nếu có data cần lưu:
>- PVC/PV.yaml => Ví dụ: 03-pvc.yaml / 03-pv.yaml

> Nếu có domain/https:
>- Ingress.yaml => Ví dụ: 06-ingress.yaml

---
> Cấu trúc thư mục gợi ý:
```
k8s/
  base/
    00-namespace.yaml
    01-configmap.yaml
    02-secret.yaml
    03-deployment.yaml
    04-service.yaml
    05-ingress.yaml
  overlays/
    dev/
    prod/
```

> Hoặc
```
k8s/
  dev/
  prod/
```

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

### 1.3 Kiểm tra các nodes đang hoạt động (Chỉ xem được trên Master)

```
kubectl get nodes -o wide
```

### 1.4 Kiểm tra Pod đang hoạt động (Chỉ xem được trên Master)

```
kubectl get pods -A -o wide
```

### 1.5 Kiểm tra Service đang hoạt động (Chỉ xem được trên Master)

```
kubectl get svc -A -o wide
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















