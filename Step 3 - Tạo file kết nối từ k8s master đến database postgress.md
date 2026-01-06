HƯỚNG DẪN TẠO FILE KẾT NỐI TỪ KUBERNETES MASTER ĐẾN DATABASE POSTGRESQL
----

> Kiểm tra đã có images chưa?

```bash
ubuntu@k8s-master:~/k8s-qlts/base/app$ docker images
REPOSITORY             TAG       IMAGE ID       CREATED       SIZE
toandnseta/qlts-cicd   latest    992b6008dfde   3 weeks ago   180MB
```

## 1. Mục đích

Tài liệu này mô tả các bước cấu hình **file kết nối Database (`.env`)**, tạo **Secret trong Kubernetes**, 
và sử dụng Secret đó để **kết nối ứng dụng chạy trong Kubernetes Cluster tới PostgreSQL Database** đang chạy bằng Docker trên server bên ngoài.

## 2. Mô hình hệ thống

- Kubernetes Cluster
  - Master Node
  - Worker Node
- Ứng dụng backend (NodeJS) chạy trong Pod
- PostgreSQL chạy bằng Docker trên server Ubuntu độc lập
Luồng kết nối:

```bash
Pod (App) → Service Network → DB Server (Docker PostgreSQL)
```

## 3. Thông tin môi trường
### 3.1. Thông tin Database Server

| Thành phần   | Giá trị          |
| ------------ | ---------------- |
| DB Server IP | `192.168.80.152` |
| DB Port      | `5432`           |
| DB Name      | `qlts_assets`    |
| DB User      | `postgres`       |
| DB Password  | `password`       |


## 4. Tạo file kết nối Database (`connect-db.env`)

Trên **Kubernetes Master Node**, tạo file cấu hình môi trường:

```bash
nano connect-db.env
```

Nội dung file:

```bash
# Database Configuration
DB_HOST=192.168.80.152
DB_PORT=5432
DB_NAME=qlts_assets
DB_USER=postgres
DB_PASSWORD=password

# JWT Secret
JWT_SECRET=your_jwt_secret_key_here

# Server Configuration
PORT=5000
NODE_ENV=production
```

## 5. Tạo Namespace cho ứng dụng

```bash
kubectl apply -f namespace.yaml
```

File `namespace.yaml`:

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: qlts
```

## 6. Tạo Kubernetes Secret từ file kết nối

Kubernetes **không sử dụng trực tiếp file** `.env`, mà cần chuyển đổi sang **Secret**.

Thực hiện lệnh:

```bash
kubectl create secret generic qlts-env \
  --from-env-file=connect-db.env \
  -n qlts
```

**Kiểm tra Secret đã tạo**

```bash
kubectl get secret qlts-env -n qlts
```

## 7. Sử dụng Secret trong Deployment

Trong file `deployment.yaml`, khai báo Secret để inject biến môi trường vào container:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qlts-app
  namespace: qlts
spec:
  replicas: 2
  selector:
    matchLabels:
      app: qlts-app
  template:
    metadata:
      labels:
        app: qlts-app
    spec:
      containers:
      - name: app
        image: toandnseta/qlts-cicd:latest
        ports:
        - containerPort: 5000
        envFrom:
        - secretRef:
            name: qlts-env
```

## 8. Deploy ứng dụng

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

## 9. Kiểm tra kết nối Database

### 9.1. Kiểm tra Pod

```bash
kubectl get pods -n qlts
```

Trạng thái đúng:

```bash
READY   STATUS
1/1     Running
```

### 9.2. Kiểm tra biến môi trường trong Pod

```bash
kubectl exec -n qlts -it <pod-name> -- sh
```

```bash
env | grep DB_
```

### 9.3. Kiểm tra log ứng dụng

```bash
kubectl logs -n qlts deploy/qlts-app
```

> Nếu kết nối DB thành công, ứng dụng sẽ hoạt động bình thường.

## 10. Lưu ý quan trọng

- PostgreSQL Docker phải expose port 5432:

```bash
0.0.0.0:5432 -> 5432
```

- PostgreSQL cho phép kết nối từ xa (listen_addresses='*')
- Firewall server DB phải mở port 5432
- Không commit file .env chứa password lên Git

## 11. Tổng kết

Quy trình kết nối Database từ Kubernetes gồm các bước chính:
1. Tạo file .env chứa thông tin DB
2. Convert file .env thành Kubernetes Secret
3. Inject Secret vào Deployment
4. Pod sử dụng biến môi trường để kết nối DB

Tài liệu này giúp đảm bảo việc cấu hình bảo mật, chuẩn Kubernetes và dễ mở rộng trong môi trường production.

-----------

## 12. Chuẩn hóa Secret & ConfigMap (Best Practice Kubernetes)

### 12.1. Mục tiêu
- Tách thông tin nhạy cảm (password, JWT) ra khỏi file cấu hình thường
- Tuân thủ best practice Kubernetes:
  - **Secret** → dữ liệu bí mật
  - **ConfigMap** → cấu hình không nhạy cảm
- Dễ bảo trì, dễ thay đổi, an toàn khi dùng Git

### 12.2. Phân loại cấu hình

> 🔐 Secret (nhạy cảm)

| Biến        | Lý do       |
| ----------- | ----------- |
| DB_PASSWORD | Mật khẩu DB |
| JWT_SECRET  | Khóa ký JWT |

> ⚙️ ConfigMap (không nhạy cảm)

| Biến     |
| -------- |
| DB_HOST  |
| DB_PORT  |
| DB_NAME  |
| DB_USER  |
| PORT     |
| NODE_ENV |


### 12.3. Tạo file ConfigMap (`configmap-db.env`)

Trên **Kubernetes Master**, tạo file:

```bash
nano configmap-db.env
```

Nội dung:

```bash
DB_HOST=192.168.80.152
DB_PORT=5432
DB_NAME=qlts_assets
DB_USER=postgres
PORT=5000
NODE_ENV=production
```

### 12.4. Tạo file Secret (`secret-db.env`)

```bash
vi secret-db.env
```

Nội dung:

```bash
DB_PASSWORD=password
JWT_SECRET=your_jwt_secret_key_here
```

> ⚠️ KHÔNG commit file này lên Git


### 12.5. Tạo ConfigMap trong Kubernetes

```bash
kubectl create configmap qlts-config \
  --from-env-file=configmap-db.env \
  -n qlts
```

**Kiểm tra**

```bash
kubectl get configmap qlts-config -n qlts
```

### 12.6. Tạo Secret trong Kubernetes

```bash
kubectl create secret generic qlts-secret \
  --from-env-file=secret-db.env \
  -n qlts
```

**Kiểm tra**

```bash
kubectl get secret qlts-secret -n qlts
```

### 12.7. Cập nhật Deployment sử dụng ConfigMap + Secret

**File `deployment.yaml` (chuẩn production)**

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qlts-app
  namespace: qlts
spec:
  replicas: 2
  selector:
    matchLabels:
      app: qlts-app
  template:
    metadata:
      labels:
        app: qlts-app
    spec:
      containers:
      - name: app
        image: toandnseta/qlts-cicd:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: qlts-config
        - secretRef:
            name: qlts-secret
```

### 12.8. Apply cấu hình mới

```bash
kubectl apply -f deployment.yaml
kubectl rollout restart deployment qlts-app -n qlts
```

### 12.9. Kiểm tra trong Pod

```bash
kubectl exec -n qlts -it <pod-name> -- sh
```

```bash
env | grep DB_
env | grep JWT
```

> Kết quả mong đợi:

```bash
DB_HOST=192.168.80.152
DB_PORT=5432
DB_NAME=qlts_assets
DB_USER=postgres
DB_PASSWORD=password
JWT_SECRET=your_jwt_secret_key_here
```

### 12.10. So sánh trước & sau chuẩn hóa

| Tiêu chí            | Trước | Sau |
| ------------------- | ----- | --- |
| Bảo mật             | ❌     | ✅   |
| Tách cấu hình       | ❌     | ✅   |
| Dễ thay đổi         | ⚠️    | ✅   |
| Dùng cho production | ❌     | ✅   |
| Phù hợp CI/CD       | ⚠️    | ✅   |


### 12.11. Best Practice khuyến nghị

- ❌ Không commit .env chứa mật khẩu
- ✅ Dùng Secret cho dữ liệu nhạy cảm
- ✅ Dùng ConfigMap cho cấu hình thường
- ✅ Mỗi môi trường (dev/stg/prod) → 1 bộ ConfigMap + Secret
- ✅ Rotate Secret bằng rollout restart

### 12.12. Sơ đồ luồng chuẩn hóa

```bash
ConfigMap ─┐
           ├─> Pod → Application → PostgreSQL
Secret ────┘
```

### 12.13. Tổng kết

Chuẩn hóa Secret + ConfigMap giúp:
- Tăng bảo mật
- Dễ mở rộng môi trường
- Đúng chuẩn Kubernetes production
- Phù hợp CI/CD và audit

