Hướng dẫn tạo cụm k8s bằng terraform trên GCP
-----------

> 🎯 MỤC TIÊU

| VM             | Vai trò                                                      |
| -------------- | ------------------------------------------------------------ |
| `k8s-master`   | Kubernetes Master                                            |
| `k8s-worker-1` | Kubernetes Worker                                            |
| `k8s-db`       | PostgreSQL (Docker)                                          |
| `k8s-cicd`     | CI/CD (GitLab Runner / Jenkins / GitHub Actions self-hosted) |

> Tất cả:
> - Chung **1 VPC riêng**
> - Chung **1 subnet**
> - SSH bằng **key Terraform tạo**
> - Có **external IP** (phục vụ lab + Cloudflare)

📁 CẤU TRÚC THƯ MỤC TERRAFORM (KHUYÊN DÙNG)

```bash
gcp-k8s-lab/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── keys/
```

## 1. Tạo file `main.tf`

```bash
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = ">= 5.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = ">= 4.0"
    }
    local = {
      source  = "hashicorp/local"
      version = ">= 2.0"
    }
  }
}

########################################
# Provider
########################################
provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

########################################
# SSH Key
########################################
resource "tls_private_key" "ssh_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "local_file" "private_key" {
  filename        = "${path.module}/keys/${var.private_key_filename}"
  content         = tls_private_key.ssh_key.private_key_pem
  file_permission = "0600"
}

########################################
# VPC + Subnet
########################################
resource "google_compute_network" "vpc" {
  name                    = var.vpc_name
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "subnet" {
  name          = var.subnet_name
  ip_cidr_range = var.subnet_cidr
  region        = var.region
  network       = google_compute_network.vpc.id
}

########################################
# Firewall
########################################
resource "google_compute_firewall" "allow_common" {
  name    = "allow-ssh-http-https"
  network = google_compute_network.vpc.name

  allow {
    protocol = "tcp"
    ports    = ["22", "80", "443", "5432"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["k8s-node", "db", "cicd"]
}


########################################
# Firewall – ICMP (ping) – internal only
########################################
resource "google_compute_firewall" "allow_icmp_internal" {
  name    = "allow-icmp-internal"
  network = google_compute_network.vpc.name

  direction = "INGRESS"

  allow {
    protocol = "icmp"
  }

  source_ranges = [var.subnet_cidr]

  target_tags = [
    "k8s-control-plane",
    "k8s-node"
  ]
}

################################################
# Firewall – Kubernetes API Server (master only)
################################################
resource "google_compute_firewall" "allow_k8s_apiserver" {
  name    = "allow-k8s-apiserver"
  network = google_compute_network.vpc.name

  direction = "INGRESS"

  allow {
    protocol = "tcp"
    ports    = ["6443"]
  }

  source_ranges = [var.subnet_cidr]

  target_tags = ["k8s-control-plane"]
}

########################################
# Firewall – Kubelet (master ↔ worker)
########################################
resource "google_compute_firewall" "allow_kubelet" {
  name    = "allow-kubelet"
  network = google_compute_network.vpc.name

  direction = "INGRESS"

  allow {
    protocol = "tcp"
    ports    = ["10250"]
  }

  source_ranges = [var.subnet_cidr]

  target_tags = [
    "k8s-control-plane",
    "k8s-node"
  ]
}

#######################################################
# Firewall – Control plane internal ports (master only)
#######################################################
resource "google_compute_firewall" "allow_control_plane_internal" {
  name    = "allow-control-plane-internal"
  network = google_compute_network.vpc.name

  direction = "INGRESS"

  allow {
    protocol = "tcp"
    ports = [
      "10257", # controller-manager
      "10259"  # scheduler
    ]
  }

  source_ranges = [var.subnet_cidr]

  target_tags = ["k8s-control-plane"]
}

############################################
# Firewall – NodePort services (worker only)
############################################
resource "google_compute_firewall" "allow_nodeport" {
  name    = "allow-nodeport"
  network = google_compute_network.vpc.name

  direction = "INGRESS"

  allow {
    protocol = "tcp"
    ports    = ["30000-32767"]
  }

  source_ranges = [var.subnet_cidr]

  target_tags = ["k8s-node"]
}

###########################################
# Firewall – Calico VXLAN (worker ↔ worker)
###########################################
resource "google_compute_firewall" "allow_calico_vxlan" {
  name    = "allow-calico-vxlan"
  network = google_compute_network.vpc.name

  direction = "INGRESS"

  allow {
    protocol = "udp"
    ports    = ["8472"]
  }

  source_ranges = [var.subnet_cidr]

  target_tags = ["k8s-node"]
}

########################################
# VM Instances (FOR_EACH)
########################################
resource "google_compute_instance" "vm" {
  for_each = var.instances

  name         = each.key
  machine_type = each.value.machine_type
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = var.instance_image
      size  = each.value.disk_size
      type  = "pd-balanced"
    }
  }

  network_interface {
    network    = google_compute_network.vpc.id
    subnetwork = google_compute_subnetwork.subnet.id
    access_config {}
  }

  metadata = {
    ssh-keys = "ubuntu:${tls_private_key.ssh_key.public_key_openssh}"
  }

  tags = each.value.tags
}

```

## 2. Tạo file `outputs.tf`

```bash
output "vm_external_ips" {
  value = {
    for k, v in google_compute_instance.vm :
    k => v.network_interface[0].access_config[0].nat_ip
  }
}

output "ssh_private_key_path" {
  value = local_file.private_key.filename
}

output "ssh_example" {
  value = {
    for k, v in google_compute_instance.vm :
    k => "ssh -i ${local_file.private_key.filename} ubuntu@${v.network_interface[0].access_config[0].nat_ip}"
  }
}
```

## 3. Tạo file `terraform.tfvars`

```bash
project_id = "ubuntu-test-prod"

instances = {
  k8s-master = {
    machine_type = "e2-small"
    disk_size    = 50
    tags         = ["k8s-control-plane"]
  }

  k8s-worker-1 = {
    machine_type = "e2-small"
    disk_size    = 50
    tags         = ["k8s-node"]
  }

  k8s-db = {
    machine_type = "e2-small"
    disk_size    = 30
    tags         = ["db"]
  }

  k8s-cicd = {
    machine_type = "e2-small"
    disk_size    = 30
    tags         = ["cicd"]
  }
}
```

## 4. Tạo file `variables.tf`

```bash
variable "project_id" {}
variable "region" {
  default = "asia-southeast1"
}
variable "zone" {
  default = "asia-southeast1-b"
}

variable "vpc_name" {
  default = "k8s-lab-vpc"
}

variable "subnet_name" {
  default = "k8s-lab-subnet"
}

variable "subnet_cidr" {
  default = "10.10.0.0/16"
}

variable "instance_image" {
  default = "ubuntu-os-cloud/ubuntu-2204-lts"
}

variable "private_key_filename" {
  default = "k8s-lab.pem"
}

########################################
# VM definitions
########################################
variable "instances" {
  type = map(object({
    machine_type = string
    disk_size    = number
    tags         = list(string)
  }))
}
```

## 5. Tạo thư mục `keys` để lưu file .pem

## 6. Khởi chạy

```bash
terraform init
terraform plan
terraform apply
```

## 7. Sử dụng SSH để kết nối instance GCP

> [Tham khảo link](https://github.com/dangtoan2205/google-cloud/blob/main/S%E1%BB%AD%20d%E1%BB%A5ng%20terraform%20full%20%2B%20s%E1%BB%AD%20d%E1%BB%A5ng%20ssh%20key%20t%E1%BB%AB%20google%20cloud.md)
