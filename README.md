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


## 🧩 Giải thích chi tiết

### **Webservice & Sidekiq**
- **Loại Kubernetes**:  
  `Deployment`  
  *→ Sử dụng Deployment vì không yêu cầu lưu trữ dữ liệu liên tục giữa các lần khởi động lại*

- **Storage**:  
  Không cần Persistent Volume  
  *→ Dữ liệu được lưu tạm thời trong memory hoặc volume ephemeral*

---

### **Gitaly/PostgreSQL/Redis**
- **Loại Kubernetes**:  
  `StatefulSet`  
  *→ Đảm bảo duy trì ổn định:*  
  - Network identity (hostname cố định)  
  - Thứ tự triển khai nghiêm ngặt  
  - Persistent Storage

- **Storage**:  
  ```yaml
  storageClass: "gp3"
  size: "500Gi" # Gitaly
  size: "100Gi"  # PostgreSQL
  size: "50Gi"   # Redis


