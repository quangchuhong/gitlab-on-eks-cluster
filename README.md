# GitLab on Amazon EKS

## Architecture

```text
Overview 

[EKS Cluster]
├── GitLab Webservice (UI/API)
├── GitLab Sidekiq (Background Jobs)
├── Gitaly (Git Repositories)
├── PostgreSQL (Database)
├── Redis (Cache/Queues)
└── CI/CD Runners (Auto-scaling)

kiến trúc triển khai GitLab trên Amazon EKS

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
---
## 2. Gitlab runner 
#### 2.1. Các loại Runner trong GitLab:
- Instance-level runner (Shared runner)
    - Áp dụng cho mọi project trong GitLab instance.
- Group-level runner: 
    - Áp dụng cho tất cả project trong 1 group (và sub-group).
- Project-level runner (Specific runner)
    - Áp dụng cho 1 project duy nhất.
- Khi job chạy, GitLab sẽ:
    - Ưu tiên project runner, rồi đến group runner, rồi mới đến shared runner,
    - Kết hợp với tags để chọn đúng runner.
      
#### 2.2. Chiến lược phân tách nhiều runner
- Tách theo group/phòng ban
  - Ví dụ:
    - Group: cloudops → runner riêng
    - Group: devops → runner riêng
    - Group: appops → runner riêng

- Cách làm:
  - Vào từng Group → Settings → CI/CD → Runners.
    - Lấy Group registration token.
- Cài runner với token đó:
```bash
gitlab-runner register \
  --url http://gitlab.gitlabonlinecom.click/ \
  --registration-token <GROUP_TOKEN_CLOUDOPS> \
  --description "cloudops-runner" \
  --executor kubernetes \
  --tag-list "cloudops,eks,k8s"

```
Hoặc với Helm values (K8s):
```bash
gitlabUrl: "http://gitlab.gitlabonlinecom.click/"
runnerRegistrationToken: "<GROUP_TOKEN_CLOUDOPS>"
runners:
  executor: "kubernetes"
  tags: "cloudops,eks,k8s"

```
  Lặp lại cho group khác (devops, appops, …) với token + tags khác.

***Kết quả***:
- Project trong group `cloudops` sẽ thấy `cloudops-runner` là “group runner”.
- Project ngoài group này không dùng được runner đó.
---
#### 2.3. Nguyên tắc chọn runner bằng tags
- Runner có danh sách tag: tags: "cloudops,eks,k8s".
- Job có tags: [cloudops] hoặc tags: [cloudops, eks] → job sẽ được assign vào runner đó.
- Nếu job không khai báo tags, mà runner lại có tags → job sẽ không chạy trên runner đó.
  
***Vì vậy***:

- Nếu muốn project A chỉ dùng runner A → job phải gắn tag mà chỉ runner A có.
- Nếu muốn chia loại workload → gán tag khác nhau cho job.
  
YAML ví dụ trong project DevOps:
```bash
stages:
  - build
  - test
  - docker
  - scan

# Job build/test Java dùng runner group DevOps
maven-build:
  stage: build
  tags: [devops]
  image: maven:3.9-eclipse-temurin-17
  script: ...

# Job docker build dùng runner docker-build
docker-build:
  stage: docker
  tags: [docker-build]
  image: gcr.io/kaniko-project/executor:v1.23.0
  script: ...

# Job scan dùng runner security
trivy-scan:
  stage: scan
  tags: [security]
  image: aquasec/trivy:0.55.0
  script: ...

```
---
#### 2.4. Khai báo nhiều gitlab runner
Không có cách “khai báo nhiều runner khác nhau” trong một block gitlab-runner: của chart GitLab; cách chuẩn là:

- GitLab (chart gitlab/gitlab) là 1 release riêng.
- Mỗi GitLab Runner là 1 release Helm riêng của chart gitlab/gitlab-runner, với values/token/tags khác nhau.

Tức là: nhiều runner = nhiều Helm release gitlab-runner, không nhét hết vào 1 values.yaml của GitLab.

Dưới đây là cách làm cụ thể.

- Runner 1 – cho group CloudOps:
`values-runner-cloudops.yaml`:
```bash
gitlabUrl: "http://gitlab.gitlabonlinecom.click/"
runnerRegistrationToken: "<GROUP_TOKEN_CLOUDOPS>"

runners:
  executor: "kubernetes"
  namespace: "gitlab"
  tags: "cloudops,eks,k8s"
  image: "python:3.11-slim"
  privileged: false
  concurrent: 10

serviceAccount:
  create: true
  name: "gitlab-runner-cloudops"

```
Cài:
```bash
helm upgrade --install gitlab-runner-cloudops gitlab/gitlab-runner \
  -n gitlab \
  -f values-runner-cloudops.yaml

```
- Runner 2 – cho group DevOps (ví dụ cần build Maven + Docker/Kaniko):
`values-runner-devops.yaml`:
```bash
gitlabUrl: "http://gitlab.gitlabonlinecom.click/"
runnerRegistrationToken: "<GROUP_TOKEN_DEVOPS>"

runners:
  executor: "kubernetes"
  namespace: "gitlab"
  tags: "devops,maven17,docker-build"
  image: "maven:3.9-eclipse-temurin-17"
  privileged: true   # nếu còn dùng DinD; nếu chỉ dùng Kaniko thì có thể false
  concurrent: 10

serviceAccount:
  create: true
  name: "gitlab-runner-devops"
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::<account-id>:role/eks-gitlab-runner-devops"

```
Cài:
```bash
helm upgrade --install gitlab-runner-devops gitlab/gitlab-runner \
  -n gitlab \
  -f values-runner-devops.yaml

```
- Runner 3 – cho project đặc biệt (ví dụ maven17 riêng):
`values-runner-maven17.yaml`:
```bash
gitlabUrl: "http://gitlab.gitlabonlinecom.click/"
runnerRegistrationToken: "<PROJECT_TOKEN_MAVEN17>"

runners:
  executor: "kubernetes"
  namespace: "gitlab"
  tags: "maven17"
  image: "maven:3.9-eclipse-temurin-17"
  privileged: false
  concurrent: 5

serviceAccount:
  create: true
  name: "gitlab-runner-maven17"

```
Cài:
```bash
helm upgrade --install gitlab-runner-maven17 gitlab/gitlab-runner \
  -n gitlab \
  -f values-runner-maven17.yaml

```
***Tóm tắt***:

- Nhiều runner = nhiều Helm release của chart gitlab/gitlab-runner:
  - gitlab-runner-cloudops
  - gitlab-runner-devops
  - gitlab-runner-maven17
  - ...
    
- Mỗi release:
  - Có runnerRegistrationToken riêng (group/project/instance).
  - Có tags riêng để job chọn đúng runner.
  - Có serviceAccount + IAM role riêng (nếu cần phân quyền AWS khác nhau).
    
Anh chỉ cần tạo thêm values-runner-*.yaml cho từng loại runner, rồi helm upgrade --install như trên.
