---
title: "Workshop"
date: 2026-07-30
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Triển khai ứng dụng Đặt sân thể thao trên AWS

#### Mục đích

Workshop này trình bày **cách chúng tôi triển khai ứng dụng Court Booking trên AWS** — từ một account trống đến một hệ thống chạy thực tế. Đây là phần thực hành tương ứng với thiết kế ở [2.1 Kiến trúc](../2-proposal/2.1-architecture/): mỗi resource được tạo ở đây đều ánh xạ tới một thành phần trong kiến trúc đó.

Stack được triển khai:

- **Backend** — FastAPI (Python 3.12) trên EC2 phía sau Application Load Balancer, deploy qua GitHub Actions → CodeDeploy
- **Database** — Amazon RDS (PostgreSQL 16), schema quản lý bằng Alembic migration
- **Authentication** — Amazon Cognito (user pool + group đóng vai trò role), được backend bao bọc (wrap)
- **Frontend** — SPA React + Vite trên AWS Amplify Hosting (CI/CD tích hợp sẵn)
- **Configuration** — SSM Parameter Store là nguồn duy nhất cho config/secret lúc runtime

#### Mô hình hai account

Dự án chạy trên **hai AWS account**, chỉ "gặp nhau" ở tầng control plane (IAM, DNS, TLS) — không có VPC peering, vì chỉ có đúng một workload VPC:

| Account | Chứa |
| --- | --- |
| **Workload account** (Thanh) | VPC, EC2/ALB, RDS, Cognito, Amplify, S3, CodeDeploy |
| **DNS / guard account** (Hiếu) | Route 53 hosted zone, bản ghi validation cho ACM, guardrail chi phí |

Mọi thao tác CLI đều nhắm vào workload account thông qua **AssumeRole** liên account (`--profile thanh`), thiết lập ở mục 5.1. Toàn bộ dự án dùng một môi trường **`dev`** duy nhất (SSM path `/court-booking/dev/`, CodeDeploy group `dev`).

#### Thứ tự triển khai

Các mục đi theo đúng thứ tự phụ thuộc thực tế — mục trước mở khóa mục sau:

```
5.1 Danh tính (IAM · MFA · OIDC)      ← ai được phép dựng hạ tầng & deploy
      → 5.2 CI pipeline (cổng chất lượng cho mỗi PR)
      → 5.3 Network + RDS             ← chuỗi Security Group, database
      → 5.4 Config & secret (SSM)     ← cách ứng dụng nhận cấu hình
      → 5.5 Cognito                   ← nguồn xác thực
      → 5.6 Deploy backend            ← EC2 + CodeDeploy walking skeleton
      → 5.7 Frontend (Amplify)        ← nối SPA với API
      → 5.8 Dọn dẹp tài nguyên
```

#### Nội dung

{{< children />}}
