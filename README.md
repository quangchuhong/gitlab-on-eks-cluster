# GitLab on Amazon EKS

## Architecture

```text
#- Overview

[EKS Cluster]
├── GitLab Webservice (UI/API)
├── GitLab Sidekiq (Background Jobs)
├── Gitaly (Git Repositories)
├── PostgreSQL (Database)
├── Redis (Cache/Queues)
└── CI/CD Runners (Auto-scaling)

#- kiến trúc triển khai GitLab trên Amazon EKS 
+-------------------------------+
|          AWS ALB              |
|  (HTTPS 443, Internet-facing) |
+---------------+---------------+
                |
                v
+---------------+-----------------------------------+
|         EKS Cluster                             |
| +---------------------+  +-------------------+  |
| | GitLab Webservice   |  | GitLab Sidekiq    |  |
| | (Deployment)        |  | (Deployment)      |  |
| +----------+----------+  +---------+---------+  |
|            |                       |             |
| +----------v-----------------------+----------+  |
| |              Shared Services                |  |
| | +--------------+  +--------------+  +-----+ |  |
| | | PostgreSQL   |  | Redis        |  |Gitaly| |  |
| | | (StatefulSet)|  | (StatefulSet)|  |(SS) | |  |
| | +--------------+  +--------------+  +-----+ |  |
| +-----------------------------------------------+  |
|                                                    |
| +------------------------------------------------+ |
| | AWS EBS (gp3)                                 | |
| | - PostgreSQL PV (100Gi)                      | |
| | - Redis PV (50Gi)                            | |
| | - Gitaly PV (500Gi)                          | |
| +------------------------------------------------+ |
+----------------------------------------------------+
```
### Thành phần chính

| Thành phần       | Mục đích                     | Loại Kubernetes          | Storage                  |
|------------------|------------------------------|--------------------------|--------------------------|
| **Webservice**   | GitLab UI & API              | Deployment               | Không lưu trữ (Ephemeral) |
| **Sidekiq**      | Xử lý background jobs        | Deployment               | Không lưu trữ (Ephemeral) |
| **Gitaly**       | Quản lý Git repositories     | StatefulSet              | EBS gp3 (500Gi+)          |
| **PostgreSQL**   | Database chính               | StatefulSet hoặc RDS     | EBS gp3 (100Gi+)          |
| **Redis**        | Cache & Queues               | StatefulSet hoặc ElastiCache | EBS gp3 (50Gi+)   |
| **ALB Ingress**  | Quản lý traffic HTTP/HTTPS   | Ingress Controller        | Không cần storage        |


## 🔧 Các loại Job chính

| Loại Job                  | Ví dụ cụ thể                          | Mô tả                              |
|---------------------------|---------------------------------------|------------------------------------|
| **Email Notifications**   | Gửi email thông báo Merge Request     | Gửi thông báo qua SMTP/SendGrid    |
| **CI/CD Pipelines**       | Chạy job build/test/deploy            | Xử lý các bước trong pipeline      |
| **Repository Management** | Xử lý Git hooks                       | Đồng bộ repository mirrors         |
| **System Maintenance**    | Dọn dẹp log, backup database          | Tự động xóa data cũ theo lịch      |
| **Webhooks & Integrations**| Gửi request tới Slack/Jira           | Kích hoạt integration khi có event |
| **User Activities**       | Update user activity analytics        | Thống kê hoạt động người dùng      |


## Redis Configuration for GitLab on EKS

### 🔑 **Key Roles of Redis**
| Role                  | Description                                                                 | Example Use Cases                  |
|-----------------------|-----------------------------------------------------------------------------|------------------------------------|
| **Cache Layer**       | Tăng tốc truy cập bằng lưu kết quả thường dùng                              | API response, HTML fragments       |
| **Background Jobs**   | Quản lý hàng đợi công việc (Sidekiq)                                        | CI/CD pipelines, Email alerts      |
| **Session Storage**   | Lưu phiên đăng nhập người dùng                                              | User authentication sessions       |
| **Rate Limiting**     | Chống spam và quá tải API                                                   | Giới hạn API requests              |
| **Real-time Features**| Hỗ trợ tính năng real-time                                                  | Live MR updates, Websocket events  |

## Background Jobs Queue

**Công cụ**: Sidekiq (dựa trên Redis)  

### Loại job điển hình:

| Loại Job       | Ví dụ                   | Mô tả                              |
|----------------|-------------------------|------------------------------------|
| **CI/CD**      | Chạy pipeline, build logs | Xử lý các job CI/CD song song, CI Job logs streaming     |
| **Email**      | Gửi thông báo Merge Request | Gửi email qua SMTP                |
| **Repository** | Mirror repositories     | Đồng bộ Git repositories từ remote |
| **System**     | Cleanup logs, backups   | Dọn dẹp hệ thống định kỳ            |

---

### ⚙️ **Configuration Guide**

#### **1. High Availability (Redis Sentinel)**
```yaml
# values.yaml
redis:
  enabled: true
  architecture: replication
  master:
    persistence:
      storageClass: gp3
      size: 50Gi
  sentinel:
    enabled: true
    quorum: 2
```


## 📂 Các loại dữ liệu chính

Dưới đây là các loại dữ liệu chính được lưu trữ trong **PostgreSQL** của GitLab:

| Loại dữ liệu               | Ví dụ cụ thể                                                                 |
|----------------------------|-----------------------------------------------------------------------------|
| **User Data**              | Thông tin người dùng (username, email, password hash, SSH keys)             |
| **Project Metadata**       | Tên repo, mô tả, visibility (public/private), cài đặt project               |
| **Issues & Merge Requests**| Tiêu đề, mô tả, comments, labels, assignees, trạng thái MR                  |
| **CI/CD Configurations**   | File `.gitlab-ci.yml`, pipeline schedules, variables                        |
| **Permissions & Roles**    | Nhóm (groups), thành viên, quyền truy cập (owner/developer/guest)          |
| **Webhooks & Integrations**| Cấu hình webhook (URL, events), tích hợp Jira, Slack                        |
| **Audit Logs**             | Lịch sử hoạt động (đăng nhập, thay đổi cài đặt, xóa project)              |
| **System Settings**        | Cấu hình GitLab instance (URL, email server, rate limits)                  |

---

### 🔍 Giải thích ngắn gọn:
- **PostgreSQL** đóng vai trò là **database chính** của GitLab, lưu trữ mọi metadata và cấu hình hệ thống.
- Dữ liệu Git repository thực tế **KHÔNG** lưu tại đây mà được lưu trữ riêng trong thư mục `/var/opt/gitlab/git-data` hoặc Object Storage.
- Định dạng bảng này phù hợp để làm tài liệu tham khảo nhanh khi debug hoặc tối ưu hệ thống.

---

📌 **Lưu ý quan trọng**:  
- **Password** được lưu dưới dạng hash (bcrypt)  
- **SSH keys** được mã hóa trước khi lưu vào database  
- **Audit logs** nên được rotate định kỳ để tránh tốn dung lượng


