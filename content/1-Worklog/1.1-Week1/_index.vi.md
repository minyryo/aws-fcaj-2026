---
title: "Worklog Tuần 1"
date: 2026-06-15
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---
{{% notice warning %}}
⚠️ **Lưu ý:** Các thông tin dưới đây chỉ nhằm mục đích tham khảo, vui lòng **không sao chép nguyên văn** cho bài báo cáo của bạn kể cả warning này.
{{% /notice %}}

### Mục tiêu tuần 1:

- Kết nối, làm quen với các thành viên trong First Cloud AI Journey.
- Hiểu dịch vụ AWS cơ bản, cách dùng Console & CLI.
- Nghiên cứu các dịch vụ cốt lõi sẽ dùng trong dự án đặt sân: EC2, RDS, Cognito, Lambda, API Gateway, SNS, S3 và CloudWatch.
- Bắt đầu lên kế hoạch kiến trúc hybrid cho ứng dụng đặt sân thể thao.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | --------- | ------------ | --------------- | -------------- |
| 2 | - Làm quen với các thành viên FCAJ <br> - Đọc và lưu ý các nội quy, quy định tại đơn vị thực tập <br> - Tổng quan phạm vi và mục tiêu dự án đặt sân thể thao <br> - **Họp kick-off nhóm**: trình bày và thống nhất đề xuất kiến trúc hybrid; phân công công việc cho các nhóm FE, BE và AWS Admin | 15/06/2026 | 15/06/2026 | |
| 3 | - Tìm hiểu AWS và các loại dịch vụ <br>&emsp; + Compute (EC2, Lambda) <br>&emsp; + Storage (S3, EBS) <br>&emsp; + Networking (VPC, ELB, API Gateway) <br>&emsp; + Database (RDS) <br>&emsp; + Security & Identity (Cognito, IAM) <br>&emsp; + Messaging (SNS) <br>&emsp; + Monitoring (CloudWatch) | 16/06/2026 | 16/06/2026 | <https://cloudjourney.awsstudygroup.com/> |
| 4 | - Tạo AWS Free Tier account <br> - Tìm hiểu AWS Console & AWS CLI <br> - **Thực hành:** <br>&emsp; + Tạo AWS account <br>&emsp; + Cài AWS CLI & cấu hình <br>&emsp; + Cách sử dụng AWS CLI | 17/06/2026 | 17/06/2026 | <https://cloudjourney.awsstudygroup.com/> |
| 5 | - Tìm hiểu EC2 cơ bản: <br>&emsp; + Instance types (tập trung: t3.small cho backend FastAPI) <br>&emsp; + AMI <br>&emsp; + EBS <br>&emsp; + Auto Scaling Groups <br>&emsp; + Elastic Load Balancer <br> - Các cách remote SSH vào EC2 <br> - Tìm hiểu Elastic IP <br> - Bắt đầu phác thảo sơ đồ kiến trúc hybrid cho ứng dụng đặt sân | 18/06/2026 | 19/06/2026 | <https://cloudjourney.awsstudygroup.com/> |
| 6 | - **Thực hành:** <br>&emsp; + Tạo EC2 instance <br>&emsp; + Kết nối SSH <br>&emsp; + Gắn EBS volume <br> - Xem lại và hoàn thiện bản phác thảo kiến trúc tuần 1 | 19/06/2026 | 19/06/2026 | <https://cloudjourney.awsstudygroup.com/> |

### Kết quả đạt được tuần 1:

- Hiểu AWS là gì và nắm được các nhóm dịch vụ cơ bản liên quan đến dự án đặt sân:
  - **Compute**: EC2 (cho backend FastAPI), Lambda (cho xử lý thanh toán)
  - **Storage**: S3 (ảnh sân và tài nguyên tĩnh), EBS (lưu trữ block cho EC2)
  - **Networking**: VPC, Elastic Load Balancer, API Gateway
  - **Database**: RDS PostgreSQL (tầng dữ liệu dùng chung cho cả monolith và serverless)
  - **Security & Identity**: Cognito (xác thực người dùng), IAM (phân quyền dịch vụ)
  - **Messaging**: SNS (thông báo xác nhận đặt sân)
  - **Monitoring**: CloudWatch (logging thống nhất cho EC2 và Lambda)

- Đã tạo và cấu hình AWS Free Tier account thành công.

- Làm quen với AWS Management Console và biết cách tìm, truy cập, sử dụng dịch vụ từ giao diện web.

- Cài đặt và cấu hình AWS CLI trên máy tính bao gồm:
  - Access Key
  - Secret Key
  - Region mặc định
  - Định dạng output

- Sử dụng AWS CLI để thực hiện các thao tác cơ bản như:
  - Kiểm tra thông tin tài khoản & cấu hình
  - Lấy danh sách region
  - Xem dịch vụ EC2 và các loại instance có sẵn
  - Tạo và quản lý key pair
  - Kiểm tra thông tin dịch vụ đang chạy

- Có khả năng kết nối giữa giao diện web và CLI để quản lý tài nguyên AWS song song.

- Phác thảo thiết kế kiến trúc hybrid ban đầu cho ứng dụng đặt sân:
  - Xác định hai luồng xử lý: EC2 monolith (logic đặt sân cốt lõi) và serverless (thanh toán + thông báo)
  - Chọn Amazon RDS (PostgreSQL) làm tầng dữ liệu dùng chung để ngăn race condition đặt trùng sân
  - Xác định CloudWatch là giải pháp giám sát thống nhất cho cả hai mô hình triển khai

---

### Biên bản họp nhóm — 15/06/2026

**Thành phần tham dự:** Toàn bộ thành viên nhóm FCAJ

**Quyết định kiến trúc**

Hieu trình bày đề xuất kiến trúc AWS hybrid cho ứng dụng đặt sân thể thao. Nhóm thống nhất phân chia ba tính năng cốt lõi như sau:

| Tính năng | Mô hình triển khai |
| --------- | ------------------ |
| Xác thực người dùng (Đăng ký / Đăng nhập) | Monolith (EC2) |
| Quản lý đặt sân (Tạo / Sửa / Hủy / Xem / Tìm kiếm) | Monolith (EC2) |
| Xử lý thanh toán | Serverless (Lambda + API Gateway) |

**Phân công công việc**

| Nhóm | Người phụ trách | Kết quả bàn giao |
| ---- | --------------- | ---------------- |
| Tất cả thành viên | Cả nhóm | Xem xét và góp ý đề xuất kiến trúc |
| Frontend | Nhóm FE | Nghiên cứu thị trường các app tương tự; thiết kế UX/UI hỗ trợ AI → design system (bảng màu, bộ font, icon) và danh sách màn hình theo từng tính năng |
| Backend — Quản lý đặt sân | Thanh | Tài liệu API (tên endpoint, input, output, use case) và thiết kế DB (bảng, cột, kiểu dữ liệu) |
| Backend — Xác thực người dùng | Nguyen | Tài liệu API và thiết kế DB cho tính năng xác thực |
| Backend — Thanh toán | Hieu | Tài liệu API và thiết kế DB cho tính năng thanh toán |
| Quản trị AWS | AWS Admin | Thiết lập AWS Organization, tài khoản, IAM users, roles và policies |

**Bước tiếp theo**
- Nhóm FE hoàn thành nghiên cứu thị trường và bộ design system sơ bộ trước cuối tuần 2
- Các thành viên BE hoàn thiện tài liệu API và sơ đồ DB trước cuối tuần 2
- AWS Admin chuẩn bị xong cấu trúc tài khoản cơ bản trước khi bắt đầu công việc hạ tầng ở tuần 3
