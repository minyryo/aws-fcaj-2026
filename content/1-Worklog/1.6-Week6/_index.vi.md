---
title: "Worklog Tuần 6"
date: 2026-07-20
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Mục tiêu tuần 6:

- Thiết lập **nền tảng danh tính AWS liên account** (IAM user + MFA, cross-account AssumeRole, GitHub OIDC provider, deploy role không cần key).
- Dựng **cổng chất lượng CI** (`ci.yml`) trong repo backend để mọi PR đều được test với database thật.
- **Gỡ nghẽn cho track deployment** bằng cách triển khai phần API backend còn lại — các endpoint được phân theo domain cho các thành viên, nhưng làm trước để công việc CI/CD và deploy tiến hành trên một app hoàn chỉnh.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | --------- | ------------ | --------------- | -------------- |
| 2 | Danh tính CLI: IAM user `hieu-cli` + MFA; **role liên account `court-booking-admin`** (AssumeRole vào workload account); kiểm chứng bằng `get-caller-identity` | 20/07/2026 | 20/07/2026 | [CI/CD guide §1.0](/2-proposal/2.2-deployment/) |
| 3 | **GitHub OIDC** identity provider + role least-privilege `gh-deploy-backend` / `gh-deploy-lambdas` (CI không cần key — không có AWS key trong GitHub) | 21/07/2026 | 21/07/2026 | [CI/CD guide §1.1–1.3](/2-proposal/2.2-deployment/) |
| 4–5 | **Pipeline `ci.yml`**: job lọc theo path, Postgres 16 service container, `alembic upgrade head`, ruff + pytest; sửa lỗi ruff version-drift (pin `ruff==0.15.21`) và bug state-pollution của test (auto-reseed scope session trong `conftest.py`) | 22/07/2026 | 23/07/2026 | [CI/CD guide §5](/2-proposal/2.2-deployment/) |
| 6 | Triển khai các **router backend còn lại** — auth, user profile, quản lý sân, manager dashboard, admin operations, payments — cùng các endpoint read (chi tiết sân, list-my-courts, schedule/images/blackouts, admin user list); 52 test pass, `ruff` sạch | 24/07/2026 | 24/07/2026 | [Proposal §6.1, §6.5, §6.6](/2-proposal/2.1-architecture/) |
| 7 | **Tham gia sự kiện** — FCAJ x Agentic AI Build Week (Bitexco Tower) | 25/07/2026 | 25/07/2026 | [§4.3](/4-eventparticipated/4.3-event3/) |
| CN | **Họp nhóm (26/07/2026)** — review tiến độ CI/CD + API; bàn giao lại quyền sở hữu | 26/07/2026 | 26/07/2026 | |

### Kết quả đạt được tuần 6:

- **Hoàn tất nền tảng danh tính liên account** — một danh tính setup con người (AssumeRole có MFA) cùng hai CI role không cần key qua GitHub OIDC, nên deploy không bao giờ mang AWS key tĩnh → [CI/CD guide §1](/2-proposal/2.2-deployment/).
- **Cổng chất lượng CI đã hoạt động** — mỗi PR nay chạy ruff + pytest trên một container Postgres 16 thật đã apply migration. Hai lỗi được chẩn đoán và sửa: ruff version-drift do không pin (local pass, CI fail trên bộ rule mở rộng của 0.16.0 → pin version), và test booking chập chờn do state-pollution dữ liệu seed (sửa bằng fixture reseed idempotent theo session).
- **Hoàn thiện bề mặt API backend** — toàn bộ router còn lại được triển khai và test (52 pass). Các endpoint này **thuộc quyền sở hữu của thành viên theo domain** (Admin Operations + role Cognito → Nguyen, Booking + Revenue → Thanh); tôi làm trước để công việc CI/CD và deployment tôi phụ trách không bị chặn vì chờ app hoàn chỉnh. Quyền sở hữu trả lại cho từng thành viên để review, tinh chỉnh, và tách payment-Lambda.
- **Tạo lại `openapi.json`** làm contract FE↔BE duy nhất; frontend (Danh & Hung) dùng nó để sinh type.

---

### Họp nhóm — 26/07/2026

**Thành viên tham dự:** Hiếu, Thanh, Nguyên, Danh, Hùng
**Vắng mặt:** Không

**Phần trình bày**

- **Hiếu** demo pipeline CI (ruff + pytest với Postgres ở mỗi PR), trình bày thiết lập danh tính liên account (AssumeRole MFA + OIDC deploy role), và cho thấy bề mặt API backend đã hoàn thiện cùng contract `openapi.json` tạo lại.
- **Thanh** và **Nguyên** review các endpoint domain của mình như đã triển khai, xác nhận contract khớp thiết kế §6.5 / §6.6.
- **Danh & Hùng** cho thấy frontend gọi endpoint thật sau công tắc mock/real, sinh type từ `openapi.json`.

**Quyết định & phân chia công việc**

| # | Hành động | Phụ trách | Ghi chú |
| - | --------- | --------- | ------- |
| 1 | Phụ trách track **hạ tầng AWS + deployment** tuần tới (network, RDS, Cognito, SSM, EC2 + CI/CD deploy) | Hiếu | Ưu tiên — app đã code-complete; điểm thiếu giờ là hạ tầng |
| 2 | Review + tinh chỉnh router **Admin Operations** và endpoint **role Cognito-first** theo §6.6; phụ trách tách **payment-Lambda** khi track serverless bắt đầu | Nguyên | Domain auth/admin của bạn ấy |
| 3 | Review + gia cố các endpoint **booking + revenue**; kiểm chứng exclusion constraint chống đặt trùng dưới tải concurrency | Thanh | Domain booking của bạn ấy |
| 4 | Dựng các màn hình **UI manager/admin/profile** với endpoint thật; chuẩn bị hosting Amplify | Danh & Hùng | Domain FE |

---

### Bảng thuật ngữ

| Viết tắt | Ý nghĩa |
| --- | --- |
| API | Application Programming Interface |
| AssumeRole | Hành động AWS STS lấy credential tạm cho role/account khác |
| CI/CD | Continuous Integration / Continuous Delivery |
| IAM | AWS Identity and Access Management |
| MFA | Multi-Factor Authentication |
| OIDC | OpenID Connect — dùng cho xác thực keyless GitHub → AWS |
| PR | Pull Request |
