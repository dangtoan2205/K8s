CÀI DOCKER + POSTGRESQL TRÊN `k8s-db`
-----

> 🎯 Mục tiêu
> - Cài Docker Engine trên VM k8s-db
> - Chạy PostgreSQL bằng Docker
> - Expose 5432 ra ngoài VM
> - Cho phép Kubernetes Pods (GCP) kết nối
> - Test kết nối từ k8s-master


## 0. SSH vào VM `k8s-db`

Từ máy local (nơi có Terraform key):

```bash
ssh -i keys/gcp_ssh_key.pem ubuntu@<K8S_DB_EXTERNAL_IP>
```

Kiểm tra:

```bash
hostname
# nên là: k8s-db
```

## 1. Cài Docker Engine (Ubuntu 22.04)

### 1.1 Update hệ thống

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

### 1.2 Thêm Docker official GPG key

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### 1.3 Thêm Docker repo

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 1.4 Cài Docker

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 1.5 Cho user `ubuntu` dùng docker (không cần sudo)

```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

Kiểm tra:

```bash
docker version
```

## 2. Chuẩn bị thư mục cho PostgreSQL

```bash
mkdir -p ~/k8s-db/postgres
cd ~/k8s-db/postgres
```

## 3. Tạo `docker-compose.yml` cho PostgreSQL

📄 docker-compose.yml

```bash
version: "3.9"

services:
  postgres:
    image: postgres:15-alpine
    container_name: qlts_postgres
    restart: unless-stopped

    environment:
      POSTGRES_DB: qlts_assets
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password

    ports:
      - "5432:5432"

    volumes:
      - postgres_data:/var/lib/postgresql/data

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d qlts_assets"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

## 4. Chạy PostgreSQL

```bash
docker compose up -d
```

Kiểm tra:

```bash
docker ps
```

Kết quả mong đợi:

```bash
qlts_postgres   postgres:15-alpine   Up (healthy)
```

## 5. Kiểm tra PostgreSQL đang listen đúng

### 5.1 Kiểm tra port

```bash
ss -lntp | grep 5432
```

Phải thấy:

```bash
0.0.0.0:5432
```

### 5.2 Test trong container

```bash
docker exec -it qlts_postgres psql -U postgres -d qlts_assets
```

## 6. Mở firewall GCP (BẮT BUỘC)

### 6.1 Firewall rule cho port 5432

> Hiện đang dùng rule chung (đã tạo):

```bash
allow {
  protocol = "tcp"
  ports    = ["5432"]
}
```

> 👉 Nếu chưa, bổ sung firewall:

```bash
gcloud compute firewall-rules create allow-postgres \
  --network k8s-lab-vpc \
  --allow tcp:5432 \
  --source-ranges 10.10.0.0/16
```

> 🔐 Best practice: chỉ cho subnet K8s truy cập

## 7. Test kết nối từ k8s-master

### 7.1 SSH vào k8s-master

```
ssh -i keys/gcp_ssh_key.pem ubuntu@<K8S_MASTER_IP>
```

### 7.2 Cài PostgreSQL client

```bash
sudo apt update
sudo apt install -y postgresql-client
```

### 7.3 Test TCP

```bash
nc -zv <K8S_DB_PRIVATE_IP> 5432
```

👉 Nếu OK:

```bash
Connection to ... 5432 port [tcp/postgresql] succeeded!
```

### 7.4 Test login DB

```bash
psql -h <K8S_DB_PRIVATE_IP> -U postgres -d qlts_assets
```

> Nhập password: `password`

## 8. Ghi nhớ thông tin để dùng cho K8s

| Biến        | Giá trị                   |
| ----------- | ------------------------- |
| DB_HOST     | **PRIVATE IP của k8s-db** |
| DB_PORT     | 5432                      |
| DB_NAME     | qlts_assets               |
| DB_USER     | postgres                  |
| DB_PASSWORD | password                  |

> 👉 KHÔNG dùng external IP cho DB (trừ khi debug)






