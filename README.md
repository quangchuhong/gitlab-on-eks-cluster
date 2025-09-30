# GitLab on Amazon EKS with ALB Ingress

This project provides a Terraform and Helm-based solution to deploy GitLab Community Edition (CE) on Amazon EKS with AWS ALB Ingress.

## Features
- 🚀 **Managed Kubernetes**: Amazon EKS cluster with managed node groups
- 🔒 **Secure Access**: HTTPS via ALB with ACM certificate
- 💾 **Persistent Storage**: EBS gp3 volumes for PostgreSQL, Redis, and Gitaly
- 🔄 **CI/CD Ready**: Integrated GitLab Runner (optional)
- 📊 **Monitoring**: Prometheus and Grafana setup

## Prerequisites
- AWS Account with IAM permissions
- Terraform v1.0+
- AWS CLI v2.7+
- kubectl v1.25+
- Helm v3.10+
- SSL Certificate in AWS Certificate Manager (ACM)

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
