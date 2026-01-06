Hướng dẫn Convert sang Ingress + Domain
-----------

## 14. Convert Service NodePort → Ingress + Domain (Chuẩn production)

### 14.1. Mục tiêu
- ❌ Không expose NodePort:30081
- ✅ Truy cập app qua domain
- ✅ Chuẩn bị sẵn cho HTTPS
- ✅ Phù hợp cluster on-prem (Calico)

### 14.2. Kiến trúc sau khi chuyển

```bash
Client
  ↓
Domain (qlts.local / qlts.example.com)
  ↓
Ingress Controller (NGINX)
  ↓
Service (ClusterIP)
  ↓
Pod (qlts-app)
```

### 14.3. Bước 1 – Chuyển Service sang ClusterIP

> 👉 Ingress chỉ route tới ClusterIP, KHÔNG dùng NodePort.

> 🔧 Sửa `service.yaml`

```bash
apiVersion: v1
kind: Service
metadata:
  name: qlts-svc
  namespace: qlts
spec:
  type: ClusterIP
  selector:
    app: qlts-app
  ports:
  - port: 80
    targetPort: 5000
```

Apply:

```bash
kubectl apply -f service.yaml
```

Kiểm tra:

```bash
kubectl get svc -n qlts
```

### 14.4. Bước 2 – Cài Ingress Controller (NGINX)

#### 14.4.1. Cài cho môi trường bare-metal / on-prem

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
```

#### 14.4.2. Kiểm tra Ingress Controller

```bash
kubectl get pods -n ingress-nginx
```

Phải thấy:

```bash
ingress-nginx-controller   1/1   Running
```

### 14.5. Bước 3 – Lấy IP Ingress Controller

```bash
kubectl get svc -n ingress-nginx
```

Ví dụ:

```bash
ingress-nginx-controller   NodePort   192.168.80.159
```

> 👉 IP này chính là IP worker node.

### 14.6. Bước 4 – Tạo Ingress resource

> 🔧 File `ingress.yaml`

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: qlts-ingress
  namespace: qlts
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: qlts.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: qlts-svc
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f ingress.yaml
```

### 14.7. Bước 5 – Cấu hình Domain

**🔹 Cách 1: Test nhanh bằng `/etc/hosts` (khuyên dùng trước)**

Trên **máy client:**

```bash
sudo nano /etc/hosts
```

Thêm:

```bash
192.168.80.159   qlts.local
```

> `192.168.80.159` = IP node chạy ingress-nginx

**🔹 Cách 2: DNS thật (Production)**

Tạo record:

```bash
qlts.example.com → 192.168.80.159
```

### 14.8. Bước 6 – Test truy cập

```bash
curl http://qlts.local/api/health
```

Hoặc browser:

```bash
http://qlts.local
```

> **Nếu OK → đã chuyển NodePort → Ingress thành công 🎉**

### 14.9. Bước 7 – Debug nhanh nếu lỗi

> ❌ 404

```bash
kubectl describe ingress -n qlts
```

> ❌ 502 / 504

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

> ❌ Không route

```bash
kubectl get endpoints -n qlts qlts-svc
```

Phải thấy IP pod 172.16.x.x.

### 14.10. (Khuyến nghị) Bật HTTPS sau khi HTTP OK

👉 Chỉ làm sau khi HTTP chạy ổn định
- cert-manager
- Let’s Encrypt
- TLS secret











