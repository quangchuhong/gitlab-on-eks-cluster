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
### 🏗 Thành phần chính

| Thành phần       | Mục đích                     | Loại Kubernetes          | Storage                  |
|------------------|------------------------------|--------------------------|--------------------------|
| **Webservice**   | Giao diện và API của GitLab   | Deployment               | Không lưu trữ (Ephemeral) |
| **Sidekiq**      | Xử lý công việc nền          | Deployment               | Không lưu trữ (Ephemeral) |
| **Gitaly**       | Quản lý Git repositories     | StatefulSet              | EBS gp3 (500Gi+)          |
| **PostgreSQL**   | Cơ sở dữ liệu chính          | StatefulSet hoặc RDS     | EBS gp3 (100Gi+)          |
| **Redis**        | Bộ nhớ đệm & Hàng đợi        | StatefulSet hoặc ElastiCache | EBS gp3 (50Gi+)   |
| **ALB Ingress**  | Quản lý traffic HTTP/HTTPS   | Ingress Controller        | Không cần storage        |

