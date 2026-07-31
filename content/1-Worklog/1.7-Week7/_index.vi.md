---
title: "Worklog Tuần 7"
date: 2026-07-27
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

> **Tuần hiện tại** (tính đến 31/07/2026). Các mục còn đang thực hiện được đánh dấu tương ứng.

### Mục tiêu tuần 7:

- Provision **hạ tầng AWS**: network (chuỗi security-group của VPC), RDS PostgreSQL, Cognito user pool, cấu hình SSM Parameter Store.
- **Deploy backend lên EC2** qua pipeline CI/CD.
- Thiết lập **Amplify hosting + custom domain** cho frontend.
- Chỉnh lại **diagram kiến trúc** theo góp ý của mentor và đồng bộ Proposal.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | --------- | ------------ | --------------- | -------------- |
| 3 | **Amplify hosting**: kết nối repo frontend, build env vars (`VITE_API_BASE_URL`, `VITE_USE_MOCK_API=false`), PR preview; custom domain `app.fcaj-rally.minervaph.works` | 28/07/2026 | 28/07/2026 | [Workshop §5.7](/5-workshop/5.7-frontend-amplify/) |
| 4 | **Network + data**: chuỗi SG (ALB → app → RDS) + DB subnet group; **RDS PostgreSQL 16**; **Cognito** user pool + app client + group role; **SSM parameter** (`DATABASE_URL`, `COGNITO_*`, `S3_BUCKET`, `CORS_ORIGINS`) | 29/07/2026 | 29/07/2026 | [Workshop §5.3–5.5](/5-workshop/) |
| 5 | **Compute + deploy path**: instance role EC2, launch template, ASG, ALB; CodeDeploy bị chặn do Free Tier plan của account (`SubscriptionRequiredException`) → chuyển sang deploy path **SSM Run Command**. **Diagram kiến trúc v6** vẽ lại theo góp ý mentor (VPC/subnet, NAT, Multi-AZ, luồng đánh số) | 30/07/2026 | 30/07/2026 | [Workshop §5.6](/5-workshop/5.6-backend-deploy/) |
| 6 | **Debug deployment** (OIDC trust-policy sai tên repo, biến `AWS_ACCOUNT_ID` chưa set, thiếu `court-booking.service`, lỗi thứ tự migration `127`); **Proposal §2.1 đồng bộ với diagram 22 bước**; bắt đầu deployment runbook cho Workshop (§5) | 31/07/2026 | *Đang thực hiện* | [Proposal §2.1](/2-proposal/2.1-architecture/) |
| 7–CN | **Backend deploy xanh + nối FE↔API**; họp nhóm (02/08/2026) | 01/08/2026 | *Dự kiến* | |

### Kết quả đạt được tuần 7:

- **Frontend đã live trên Amplify** tại `https://app.fcaj-rally.minervaph.works` — CI/CD kết nối git, build env vars, custom domain.
- **Đã provision hạ tầng AWS cốt lõi** — chuỗi security-group của VPC, RDS PostgreSQL 16, Cognito user pool + group, và SSM Parameter Store làm nguồn config runtime duy nhất. Đã xử lý một số ràng buộc của Free Tier plan (giới hạn backup-retention và Multi-AZ, quy tắc ký tự cho master-password RDS).
- **Dựng và debug pipeline deploy** — vì Free Tier plan của account chặn CodeDeploy, deploy backend được làm lại trên **SSM Run Command** (bundle → S3 → `ssm send-command` → install/migrate/start/validate trên instance). Đã xử lý một chuỗi lỗi lần deploy đầu: OIDC trust policy (sai tên repo), một biến GitHub chưa set, thiếu systemd unit, và migration chạy trước khi venv tồn tại. *(Dự kiến backend deploy xanh cuối tuần.)*
- **Diagram kiến trúc v6** — vẽ lại để đáp ứng góp ý mentor (VPC rõ ràng, public/private subnet, NAT, RDS Multi-AZ, ALB + ASG trên hai AZ, GitHub nằm ngoài ranh giới AWS Cloud), kèm luồng đánh số 22 bước; **phần walkthrough Proposal §2.1 được đồng bộ** khớp diagram.
- **Deployment runbook (Workshop §5)** được viết, ghi lại toàn bộ quá trình build cùng mọi lỗi và cách sửa.

---

### Họp nhóm — 02/08/2026 *(dự kiến)*

**Nội dung:** chốt backend deploy (SSM path), quyết định nối FE↔API (Amplify `/api/*` proxy vs. `api.<domain>` kèm ACM), khởi động track payment-Lambda (Nguyên), gia cố booking dưới concurrency (Thanh), review UI manager/admin (Danh & Hùng).

---

### Bảng thuật ngữ

| Viết tắt | Ý nghĩa |
| --- | --- |
| ACM | AWS Certificate Manager — cấp/quản lý chứng chỉ TLS |
| ALB | Application Load Balancer |
| ASG | Auto Scaling Group |
| CDN | Content Delivery Network |
| NAT | Network Address Translation (Gateway) — đường ra internet cho private subnet |
| RDS | Amazon Relational Database Service |
| SG | Security Group — tường lửa ảo mức instance |
| SSM | AWS Systems Manager (Parameter Store, Run Command) |
| VPC | Virtual Private Cloud |
