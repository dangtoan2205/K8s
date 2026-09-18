Tổng quan Kubernetes (K8s)

Kubernetes là nền tảng orchestration dùng để:

Deploy container
Scale hệ thống tự động
Self-healing (tự phục hồi)
Load balancing
Quản lý networking, storage, secret, config

Kiến trúc K8s được chia thành 2 phần chính:

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
1. KIẾN TRÚC TỔNG THỂ K8S

<img width="750" height="761" alt="image" src="https://github.com/user-attachments/assets/4e320896-57c1-4b34-9419-5c355ad517b2" />

<img width="770" height="717" alt="image" src="https://github.com/user-attachments/assets/e2fdfb93-c6fb-4415-af70-7f9cb4a6aecd" />

<img width="1290" height="860" alt="image" src="https://github.com/user-attachments/assets/e0bb7559-0b79-40ae-bf81-08da170f48a7" />

