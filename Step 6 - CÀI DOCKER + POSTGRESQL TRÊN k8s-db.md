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
mkdir -p ~/k8s-db/database
cd ~/k8s-db/database
```

## 3. Tạo file và các thư mục liên quan cho PostgreSQL

### 3.1📄 Cấu trúc thư mục

```bash
k8s-db/
├── docker-compose.yml
└── database/
    └── schema.sql
```

### 3.2📄Tạo file `docker-compose.yml`

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

### 3.3 Tạo file `database/schema.sql`

```bash
-- Tạo database schema cho quản lý tài sản IT

-- Bảng nhân viên
CREATE TABLE IF NOT EXISTS employees (
    id SERIAL PRIMARY KEY,
    employee_id VARCHAR(20) UNIQUE NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    department VARCHAR(50),
    position VARCHAR(50),
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bảng loại tài sản
CREATE TABLE IF NOT EXISTS asset_types (
    id SERIAL PRIMARY KEY,
    type_name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bảng tài sản
CREATE TABLE IF NOT EXISTS assets (
    id SERIAL PRIMARY KEY,
    asset_code VARCHAR(50) UNIQUE NOT NULL,
    asset_name VARCHAR(100) NOT NULL,
    asset_type_id INTEGER REFERENCES asset_types(id),
    brand VARCHAR(50),
    model VARCHAR(50),
    serial_number VARCHAR(100),
    purchase_date DATE,
    purchase_price DECIMAL(12,2),
    status VARCHAR(20) DEFAULT 'available', -- available, assigned, maintenance, retired
    location VARCHAR(100),
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bảng lịch sử bàn giao tài sản
CREATE TABLE IF NOT EXISTS asset_assignments (
    id SERIAL PRIMARY KEY,
    asset_id INTEGER REFERENCES assets(id),
    employee_id INTEGER REFERENCES employees(id),
    assigned_date DATE NOT NULL,
    return_date DATE,
    assigned_by VARCHAR(100),
    notes TEXT,
    status VARCHAR(20) DEFAULT 'active', -- active, returned
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bảng người dùng hệ thống
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'user', -- admin, user
    employee_id INTEGER REFERENCES employees(id),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bảng quyền chi tiết cho từng user
CREATE TABLE IF NOT EXISTS user_permissions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    entity_type VARCHAR(50) NOT NULL, -- asset, employee, assignment
    can_view BOOLEAN DEFAULT true,
    can_edit BOOLEAN DEFAULT false,
    can_delete BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, entity_type)
);

-- Insert dữ liệu mẫu cho loại tài sản
INSERT INTO asset_types (type_name, description) VALUES
('Case PC', 'Thùng máy tính để bàn'),
('Màn hình', 'Monitor màn hình máy tính'),
('Bàn phím', 'Keyboard bàn phím máy tính'),
('Chuột', 'Mouse chuột máy tính'),
('Tai nghe', 'Headphone tai nghe'),
('Laptop', 'Máy tính xách tay'),
('MacBook', 'MacBook của Apple'),
('Thiết bị khác', 'Các thiết bị IT khác');

-- Bảng lịch sử thay đổi (activity logs)
CREATE TABLE IF NOT EXISTS activity_logs (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    username VARCHAR(50) NOT NULL,
    action_type VARCHAR(50) NOT NULL, -- create, update, delete, assign, return
    entity_type VARCHAR(50) NOT NULL, -- asset, employee, assignment
    entity_id INTEGER,
    entity_name VARCHAR(255),
    old_values JSONB,
    new_values JSONB,
    description TEXT,
    ip_address VARCHAR(45),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert admin user mặc định
INSERT INTO users (username, email, password_hash, role) VALUES
('admin', 'admin@company.com', '$2a$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', 'admin');

-- Insert user acc
INSERT INTO users (username, email, password_hash, role) VALUES
('acc', 'acc@company.com', '$2a$10$.iTRzZfEMKKSVL9q.M1WQOZIXKO1YPxVcnwvl1YMn5MLHu06AQUFK', 'user');

-- Tạo indexes để tối ưu hiệu suất
CREATE INDEX idx_assets_asset_code ON assets(asset_code);
CREATE INDEX idx_assets_status ON assets(status);
CREATE INDEX idx_asset_assignments_asset_id ON asset_assignments(asset_id);
CREATE INDEX idx_asset_assignments_employee_id ON asset_assignments(employee_id);
CREATE INDEX idx_asset_assignments_status ON asset_assignments(status);
CREATE INDEX idx_employees_employee_id ON employees(employee_id);
CREATE INDEX idx_activity_logs_user_id ON activity_logs(user_id);
CREATE INDEX idx_activity_logs_entity_type ON activity_logs(entity_type);
CREATE INDEX idx_activity_logs_created_at ON activity_logs(created_at);
CREATE INDEX idx_user_permissions_user_id ON user_permissions(user_id);
CREATE INDEX idx_user_permissions_entity_type ON user_permissions(entity_type);
-- Tạo database schema cho quản lý tài sản IT

-- Bảng nhân viên
CREATE TABLE IF NOT EXISTS employees (
    id SERIAL PRIMARY KEY,
    employee_id VARCHAR(20) UNIQUE NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    department VARCHAR(50),
    position VARCHAR(50),
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bảng loại tài sản
CREATE TABLE IF NOT EXISTS asset_types (
    id SERIAL PRIMARY KEY,
    type_name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bảng tài sản
CREATE TABLE IF NOT EXISTS assets (
    id SERIAL PRIMARY KEY,
    asset_code VARCHAR(50) UNIQUE NOT NULL,
    asset_name VARCHAR(100) NOT NULL,
    asset_type_id INTEGER REFERENCES asset_types(id),
    brand VARCHAR(50),
    model VARCHAR(50),
    serial_number VARCHAR(100),
    purchase_date DATE,
    purchase_price DECIMAL(12,2),
    status VARCHAR(20) DEFAULT 'available', -- available, assigned, maintenance, retired
    location VARCHAR(100),
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bảng lịch sử bàn giao tài sản
CREATE TABLE IF NOT EXISTS asset_assignments (
    id SERIAL PRIMARY KEY,
    asset_id INTEGER REFERENCES assets(id),
    employee_id INTEGER REFERENCES employees(id),
    assigned_date DATE NOT NULL,
    return_date DATE,
    assigned_by VARCHAR(100),
    notes TEXT,
    status VARCHAR(20) DEFAULT 'active', -- active, returned
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bảng người dùng hệ thống
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'user', -- admin, user
    employee_id INTEGER REFERENCES employees(id),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bảng quyền chi tiết cho từng user
CREATE TABLE IF NOT EXISTS user_permissions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    entity_type VARCHAR(50) NOT NULL, -- asset, employee, assignment
    can_view BOOLEAN DEFAULT true,
    can_edit BOOLEAN DEFAULT false,
    can_delete BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, entity_type)
);

-- Insert dữ liệu mẫu cho loại tài sản
INSERT INTO asset_types (type_name, description) VALUES
('Case PC', 'Thùng máy tính để bàn'),
('Màn hình', 'Monitor màn hình máy tính'),
('Bàn phím', 'Keyboard bàn phím máy tính'),
('Chuột', 'Mouse chuột máy tính'),
('Tai nghe', 'Headphone tai nghe'),
('Laptop', 'Máy tính xách tay'),
('MacBook', 'MacBook của Apple'),
('Thiết bị khác', 'Các thiết bị IT khác');

-- Bảng lịch sử thay đổi (activity logs)
CREATE TABLE IF NOT EXISTS activity_logs (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    username VARCHAR(50) NOT NULL,
    action_type VARCHAR(50) NOT NULL, -- create, update, delete, assign, return
    entity_type VARCHAR(50) NOT NULL, -- asset, employee, assignment
    entity_id INTEGER,
    entity_name VARCHAR(255),
    old_values JSONB,
    new_values JSONB,
    description TEXT,
    ip_address VARCHAR(45),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert admin user mặc định
INSERT INTO users (username, email, password_hash, role) VALUES
('admin', 'admin@company.com', '$2a$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', 'admin');

-- Insert user acc
INSERT INTO users (username, email, password_hash, role) VALUES
('acc', 'acc@company.com', '$2a$10$.iTRzZfEMKKSVL9q.M1WQOZIXKO1YPxVcnwvl1YMn5MLHu06AQUFK', 'user');

-- Tạo indexes để tối ưu hiệu suất
CREATE INDEX idx_assets_asset_code ON assets(asset_code);
CREATE INDEX idx_assets_status ON assets(status);
CREATE INDEX idx_asset_assignments_asset_id ON asset_assignments(asset_id);
CREATE INDEX idx_asset_assignments_employee_id ON asset_assignments(employee_id);
CREATE INDEX idx_asset_assignments_status ON asset_assignments(status);
CREATE INDEX idx_employees_employee_id ON employees(employee_id);
CREATE INDEX idx_activity_logs_user_id ON activity_logs(user_id);
CREATE INDEX idx_activity_logs_entity_type ON activity_logs(entity_type);
CREATE INDEX idx_activity_logs_created_at ON activity_logs(created_at);
CREATE INDEX idx_user_permissions_user_id ON user_permissions(user_id);
CREATE INDEX idx_user_permissions_entity_type ON user_permissions(entity_type);
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






