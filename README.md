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
