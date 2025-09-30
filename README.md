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
# Thành phần chính
Component	Mục đích	Loại Kubernetes	Storage
Webservice	GitLab UI & API	Deployment	Ephemeral
Sidekiq	Xử lý background jobs	Deployment	Ephemeral
Gitaly	Quản lý Git repositories	StatefulSet	EBS gp3 (500Gi+)
PostgreSQL	Database chính	StatefulSet/RDS	EBS gp3 (100Gi+)
Redis	Cache & Queues	StatefulSet/ElastiCache	EBS gp3 (50Gi+)
ALB Ingress	Quản lý traffic HTTP/HTTPS	Ingress Controller	-
