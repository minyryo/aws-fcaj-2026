---
title: "Worklog Tuần 5"
date: 2026-07-13
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Mục tiêu tuần 5:

- Chốt thiết kế CI/CD & quy trình triển khai (hai repo, chiến lược hai tài khoản AWS) — chi tiết tại [Bản đề xuất §2.2](/vi/2-proposal/2.2-deployment/).
- Điều chỉnh thiết kế thống nhất cho vai trò mới **Court Manager** — chi tiết tại [Bản đề xuất §2.1, §6.5](/vi/2-proposal/2.1-architecture/).
- Khởi tạo cả hai code repository và scaffold backend (FastAPI + Alembic); triển khai schema 7 bảng trên môi trường local.
- Bắt đầu kết nối frontend–backend trên local.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc                                                                                                                                                                                                                | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                                                    |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------ | --------------- | ------------------------------------------------------------------ |
| 2   | - Quyết định **chiến lược triển khai hai tài khoản** (dev trên tài khoản AWS cá nhân, prod trên tài khoản có credit) và kế hoạch phân giai đoạn (repo & CI trước, IaC/CD sau)                                            | 13/07/2026   | 13/07/2026      | [Bản đề xuất §2.2](/vi/2-proposal/2.2-deployment/)                 |
| 3   | - **Điều chỉnh thiết kế Court Manager** (§6.5): đổi tên vai trò, 10 endpoint mới, bảng `court_schedules`/`court_blackouts`, cập nhật ERD <br> - Tạo repo `court-booking-backend` & `court-booking-frontend`; scaffold BE theo hướng dẫn từng bước (§2.4); thiết lập pyenv-virtualenv | 14/07/2026   | 14/07/2026      | [Bản đề xuất §2.1](/vi/2-proposal/2.1-architecture/)               |
| 6   | - Triển khai model SQLAlchemy cho schema 7 bảng; review nhiều vòng đối chiếu §6.2 (kiểu dữ liệu, ràng buộc, nullability, index cho FK)                                                                                   | 17/07/2026   | 17/07/2026      | [Bản đề xuất §6.2](/vi/2-proposal/2.1-architecture/)               |
| 7   | - Postgres local qua Docker Compose; nối Alembic và áp dụng các migration đầu tiên (`btree_gist`, các bảng cốt lõi) <br> - Thêm CORS; kiểm tra tầng service của FE (axios client, bộ chuyển mock/real) theo đặc tả §6.1  | 18/07/2026   | 18/07/2026      | [Hướng dẫn CI/CD Part 0.5](/vi/2-proposal/2.2-deployment/)         |
| CN  | - **Họp nhóm (19/07/2026)**: demo tiến độ BE; review các API bổ sung do Nguyên đề xuất                                                                                                                                   | 19/07/2026   | 19/07/2026      |                                                                    |

### Kết quả đạt được tuần 5:

Chi tiết thiết kế đầy đủ nằm trong Bản đề xuất — danh sách này chỉ ghi nhận công việc đã làm, kèm liên kết.

- **Chốt thiết kế triển khai** — hai repo theo nhóm phụ trách, GitHub Actions điều phối Amplify / CodeDeploy / SAM, kiểm tra API contract bằng OpenAPI, và chiến lược hai tài khoản (dev/prod) → [Bản đề xuất §2.2](/vi/2-proposal/2.2-deployment/).
- **Điều chỉnh thiết kế Court Manager** — đổi tên vai trò, API 15 → 25 endpoint, schema 5 → 7 bảng, cập nhật ERD Mermaid + drawio → [Bản đề xuất §6.5](/vi/2-proposal/2.1-architecture/).
- **Cả hai repository đã hoạt động**; backend scaffold trên nhánh `chore/initial-setup`: khung FastAPI với `/api/v1/health`, model SQLAlchemy 7 bảng bám sát §6.2 (kiểm chứng bằng cấu hình mapper trực tiếp), chuỗi Alembic đã áp dụng và kiểm tra (`btree_gist` → các bảng cốt lõi, gồm exclusion constraint chống đặt trùng sân).
- **Đã kiểm tra tầng service của FE** theo đặc tả §6.1: axios client dùng chung với luồng refresh khi 401, cặp service real/mock sau cờ `VITE_USE_MOCK_API` — kế hoạch kết nối được ghi tại Hướng dẫn CI/CD Part 0.5.
- **Tài liệu cho nhóm**: README backend (thiết lập, quy ước, bảng xử lý sự cố, nhật ký thay đổi thiết kế), `alembic-working-guide.md`, và `cicd-setup-guide.md` mở rộng (phần scaffold + kết nối FE–BE).

---

### Biên bản họp nhóm — 19/07/2026

**Thành phần tham dự:** Hiếu, Thành, Nguyên, Danh, Hùng
**Vắng mặt:** Không

**Trình bày**

- **Hiếu** demo backend scaffold: cấu trúc repo, migration schema 7 bảng áp dụng trên Postgres local, và giới thiệu hướng dẫn triển khai (CI/CD hai repo, chiến lược hai tài khoản).
- **Nguyên** đề xuất 6 endpoint bổ sung mở rộng thiết kế API thống nhất (tài liệu `ADJ_APIs.md`): **Admin Operations** (hàng chờ duyệt sân, approve/reject, quản lý vai trò người dùng), **Manager Analytics** (doanh thu), và **User Profile** (xem/cập nhật — hoãn lại do độ ưu tiên thấp).

**Tóm tắt review các API bổ sung của Nguyên**

1. **Các endpoint duyệt sân của admin lấp một lỗ hổng thực sự**: §6.5 đưa ra vòng đời sân `PENDING` → `ACTIVE`/`REJECTED` nhưng chưa định nghĩa API admin nào để vận hành nó. `GET /admin/courts` (hàng chờ) + endpoint review hoàn thiện luồng này, kèm thông báo SNS đến court manager khi có quyết định.
2. **Endpoint doanh thu phục vụ dashboard của manager** đã hứa ở §6.5. Định nghĩa thống nhất: tổng hợp từ **`payments` có `status = 'SUCCESS'`** (các khoản hoàn tiền tự động bị loại) join với các sân của người gọi — không dùng tổng tiền booking — và giới hạn theo `courts.owner_id` theo quy tắc chống IDOR, kèm tham số tùy chọn `group_by=day|court` cho biểu đồ.
3. **Đổi vai trò phải đi qua Cognito trước**: `users.role` chỉ là cache — endpoint phải gọi Cognito `AdminAddUserToGroup`/`AdminRemoveUserFromGroup` rồi mới cập nhật bản ghi DB, nếu không claim trong JWT và DB sẽ lệch nhau.
4. **Chuẩn hóa cách đặt tên**: `GET/PUT /users/me` (khớp mẫu `/bookings/me` hiện có) thay cho kiểu động từ `/users/update-profile`.
5. **Cần bổ sung schema**: cột `courts.rejection_reason` (nullable) để lý do từ chối sân còn lưu lại sau khi thông báo đã gửi.

**Quyết định & bước tiếp theo**

| # | Hành động | Phụ trách |
| - | --------- | --------- |
| 1 | Chấp nhận cả 6 endpoint với các điều chỉnh nêu trên; tích hợp vào Bản đề xuất thành bản điều chỉnh §6.6 | Hiếu & Nguyên |
| 2 | Tiếp tục router đặt sân trên schema đã scaffold | Thành |
| 3 | Hoàn tất bước kiểm chứng kết nối FE–BE (health check qua CORS) và tiếp tục kết nối theo từng tính năng | Danh & Hùng |
| 4 | Hoãn các endpoint hồ sơ người dùng (ưu tiên thấp, quay lại sau các tính năng cốt lõi) | — |

---

### Bảng thuật ngữ viết tắt

| Viết tắt | Ý nghĩa |
| --- | --- |
| API | Giao diện lập trình ứng dụng — quy ước để các thành phần phần mềm giao tiếp với nhau |
| AWS | Nền tảng điện toán đám mây của Amazon |
| BE | Backend — phần phía máy chủ của ứng dụng |
| CI/CD | Tích hợp liên tục / Chuyển giao liên tục — tự động build, test và triển khai |
| CORS | Cross-Origin Resource Sharing — cơ chế trình duyệt cho phép trang web gọi API ở origin khác |
| DB | Cơ sở dữ liệu |
| ERD | Sơ đồ thực thể – quan hệ, mô hình hóa các bảng và quan hệ trong CSDL |
| FE | Frontend — phần giao diện phía người dùng của ứng dụng |
| FK | Khóa ngoại — cột tham chiếu khóa chính của bảng khác |
| IaC | Hạ tầng dưới dạng mã — khai báo hạ tầng trong tệp có quản lý phiên bản |
| IDOR | Insecure Direct Object Reference — lỗ hổng truy cập tài nguyên của người dùng khác bằng cách thao túng ID đối tượng |
| JWT | Token ký số mang thông tin danh tính (claims) |
| SNS | Dịch vụ thông báo pub/sub của AWS |
