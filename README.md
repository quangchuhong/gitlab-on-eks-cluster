# GitLab on Amazon EKS

## Architecture

kiến trúc triển khai GitLab trên Amazon EKS 
```
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



