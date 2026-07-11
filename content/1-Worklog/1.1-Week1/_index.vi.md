---
title: "Worklog Tuần 1"
date: 2026-06-15
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---

### Mục tiêu tuần 1:

- Kết nối, làm quen với các thành viên trong First Cloud AI Journey.
- Tham dự họp kick-off và thống nhất đề xuất kiến trúc AWS hybrid.
- Nghiên cứu mô hình kiến trúc monolith và serverless để hiểu cách tiếp cận hybrid.
- Đọc tổng quan các dịch vụ AWS được đề cập trong sơ đồ kiến trúc dự án.
- Tạo tài khoản AWS.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc                                                                                                                                                                                                                                                                                                                           | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - **Họp nhóm**: tham dự cuộc họp kick-off, trình bày và thống nhất đề xuất kiến trúc hybrid; phân công công việc cho các nhóm FE, BE và AWS Admin                                                                                                                                                                                   | 15/06/2026   | 15/06/2026      |                                           |
| 3   | - Nghiên cứu về kiến trúc monolith & serverless                                                                                                                                                                                                                                                                                     | 16/06/2026   | 16/06/2026      |                                           |
| 4   | - Đọc tổng quan các dịch vụ AWS được đề cập trong sơ đồ kiến trúc: <br>&emsp; + EC2, Lambda (Compute) <br>&emsp; + RDS PostgreSQL (Database) <br>&emsp; + API Gateway, ELB (Networking) <br>&emsp; + Cognito, IAM (Security & Identity) <br>&emsp; + SNS (Messaging) <br>&emsp; + S3 (Storage) <br>&emsp; + CloudWatch (Monitoring) | 17/06/2026   | 18/06/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 5   | - Tạo tài khoản AWS Free Tier                                                                                                                                                                                                                                                                                                       | 19/06/2026   | 19/06/2026      |                                           |

### Kết quả đạt được tuần 1:

- Tham dự họp kick-off nhóm; thống nhất kiến trúc AWS hybrid phân chia ứng dụng đặt sân thành luồng EC2 monolith (Xác thực + Đặt sân) và luồng serverless (Thanh toán).

- Hiểu được sự khác biệt giữa hai mô hình kiến trúc:
  - **Monolith**: triển khai đơn khối, có trạng thái, độ trễ thấp — phù hợp với logic đặt sân cốt lõi trên EC2
  - **Serverless**: hướng sự kiện, có thể mở rộng độc lập, không cần quản lý server — phù hợp với xử lý webhook thanh toán trên Lambda

- Đọc tổng quan tất cả các dịch vụ AWS trong sơ đồ kiến trúc:
  - **Compute**: EC2 (backend FastAPI), Lambda (xử lý thanh toán)
  - **Database**: RDS PostgreSQL (tầng dữ liệu dùng chung, ngăn race condition đặt trùng sân)
  - **Networking**: API Gateway (điểm nhận webhook), Elastic Load Balancer (phân phối tải cho EC2)
  - **Security & Identity**: Cognito (xác thực người dùng), IAM (phân quyền dịch vụ)
  - **Messaging**: SNS (thông báo xác nhận đặt sân)
  - **Storage**: S3 (ảnh sân và tài nguyên tĩnh)
  - **Monitoring**: CloudWatch (logging thống nhất cho EC2 và Lambda)

- Đã tạo tài khoản AWS Free Tier thành công.

---

### Biên bản họp nhóm — 20/06/2026

**Thành phần tham dự:** Hiếu, Thành, Danh (Vắng: Nguyên, Hùng)

**Quyết định kiến trúc**

Hieu trình bày đề xuất kiến trúc AWS hybrid cho ứng dụng đặt sân thể thao. Nhóm thống nhất phân chia ba tính năng cốt lõi như sau:

| Tính năng                                          | Mô hình triển khai                |
| -------------------------------------------------- | --------------------------------- |
| Xác thực người dùng (Đăng ký / Đăng nhập)          | Monolith (EC2)                    |
| Quản lý đặt sân (Tạo / Sửa / Hủy / Xem / Tìm kiếm) | Monolith (EC2)                    |
| Xử lý thanh toán                                   | Serverless (Lambda + API Gateway) |

**Phân công công việc**

| Nhóm                          | Người phụ trách | Kết quả bàn giao                                                                                                                                     |
| ----------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tất cả thành viên             | Cả nhóm         | Xem xét và góp ý đề xuất kiến trúc                                                                                                                   |
| Frontend                      | Nhóm FE         | Nghiên cứu thị trường các app tương tự; thiết kế UX/UI hỗ trợ AI → design system (bảng màu, bộ font, icon) và danh sách màn hình theo từng tính năng |
| Backend — Quản lý đặt sân     | Thanh           | Tài liệu API (tên endpoint, input, output, use case) và thiết kế DB (bảng, cột, kiểu dữ liệu)                                                        |
| Backend — Xác thực người dùng | Nguyen          | Tài liệu API và thiết kế DB cho tính năng xác thực                                                                                                   |
| Backend — Thanh toán          | Hieu            | Tài liệu API và thiết kế DB cho tính năng thanh toán                                                                                                 |
| Quản trị AWS                  | AWS Admin       | Thiết lập AWS Organization, tài khoản, IAM users, roles và policies                                                                                  |

**Bước tiếp theo**

- Nhóm FE hoàn thành nghiên cứu thị trường và bộ design system sơ bộ trước cuối tuần 2
- Các thành viên BE hoàn thiện tài liệu API và sơ đồ DB trước cuối tuần 2
- AWS Admin chuẩn bị xong cấu trúc tài khoản cơ bản trước khi bắt đầu công việc hạ tầng ở tuần 3

---

### Bảng thuật ngữ viết tắt

| Viết tắt | Ý nghĩa |
| --- | --- |
| AI | Trí tuệ nhân tạo |
| API | Giao diện lập trình ứng dụng — hợp đồng giao tiếp giữa các thành phần phần mềm |
| AWS | Nền tảng điện toán đám mây của Amazon |
| BE | Backend — phần phía máy chủ của ứng dụng |
| DB | Cơ sở dữ liệu |
| EC2 | Amazon Elastic Compute Cloud — máy chủ ảo trên AWS |
| ELB | Bộ cân bằng tải — phân phối lưu lượng vào giữa các instance |
| FE | Frontend — phần giao diện phía người dùng của ứng dụng |
| IAM | Quản lý danh tính và quyền truy cập của AWS — user, role và quyền hạn |
| RDS | Dịch vụ cơ sở dữ liệu quan hệ được quản lý của AWS |
| S3 | Dịch vụ lưu trữ đối tượng của AWS |
| SNS | Dịch vụ thông báo pub/sub của AWS |
| UI | Giao diện người dùng |
| UX | Trải nghiệm người dùng |
