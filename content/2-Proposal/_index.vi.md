---
title: "Bản đề xuất"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

Tại phần này, bạn cần tóm tắt các nội dung trong workshop mà bạn **dự tính** sẽ làm.

# Ứng dụng Đặt sân thể thao

## Kiến trúc AWS Hybrid cho hệ thống đặt sân hiệu quả và đáng tin cậy

### 1. Tóm tắt điều hành

Dự án này thiết kế và triển khai kiến trúc AWS hybrid cho ứng dụng đặt sân thể thao, cho phép người dùng tìm kiếm, xem thông tin và đặt sân trực tuyến. Logic nghiệp vụ cốt lõi chạy trên EC2 theo mô hình monolith, trong khi xử lý thanh toán và gửi thông báo được thực hiện qua luồng serverless hướng sự kiện. Cả hai luồng đều dùng chung một Amazon RDS (PostgreSQL) làm nguồn dữ liệu duy nhất, ngăn chặn các lỗi nhất quán dữ liệu như đặt trùng sân. Quản lý danh tính người dùng do Amazon Cognito đảm nhận, và sức khỏe hệ thống được giám sát tập trung qua Amazon CloudWatch.

### 2. Tuyên bố vấn đề

#### Vấn đề hiện tại

Quản lý đặt sân thủ công dễ xảy ra lỗi và khó mở rộng. Khi không có hệ thống tập trung, tình trạng đặt trùng sân rất phổ biến, xác nhận thanh toán chậm và không đáng tin cậy, đồng thời người dùng không có thông tin thời gian thực về tình trạng sân trống.

#### Giải pháp

Kiến trúc hybrid kết hợp độ tin cậy của logic ứng dụng chạy trên EC2 với tính co giãn của xử lý thanh toán serverless. Các yêu cầu từ người dùng (duyệt, tìm kiếm, đặt sân) được xử lý bởi backend FastAPI trên EC2, đứng sau Elastic Load Balancer kết hợp Auto Scaling. Sự kiện thanh toán được định tuyến qua Amazon API Gateway đến AWS Lambda, xác nhận thanh toán và ghi booking vào RDS, sau đó thông báo cho người dùng qua Amazon SNS. Ảnh sân và tài nguyên tĩnh được phục vụ từ Amazon S3.

#### Lợi ích và hoàn vốn đầu tư

- Ngăn chặn đặt trùng sân nhờ tính toàn vẹn quan hệ được đảm bảo tại tầng RDS
- Tự động mở rộng trong giờ cao điểm mà không cần dự phòng EC2 quá mức
- Giảm chi phí hạ tầng thanh toán nhờ mô hình tính phí theo lần gọi của Lambda
- Cung cấp một giao diện giám sát thống nhất cho cả hai mô hình triển khai qua CloudWatch
- Thể hiện năng lực với EC2, Lambda và API Gateway - phù hợp trực tiếp với mục tiêu tuần 3 của chương trình AWS FCJ

### 3. Tính năng cốt lõi của ứng dụng

Ứng dụng tập trung vào ba tính năng chính, mỗi tính năng được phân bổ mô hình triển khai phù hợp với đặc điểm tải của nó:

| Tính năng                                          | Mô hình triển khai                | Lý do                                                                          |
| -------------------------------------------------- | --------------------------------- | ------------------------------------------------------------------------------ |
| Xác thực người dùng (Đăng ký / Đăng nhập)          | Monolith (EC2)                    | Tần suất cao, nhạy cảm với độ trễ; hưởng lợi từ tính toán liên tục             |
| Quản lý đặt sân (Tạo / Sửa / Hủy / Xem / Tìm kiếm) | Monolith (EC2)                    | Logic nghiệp vụ cốt lõi yêu cầu tính toàn vẹn quan hệ và truy vấn phức tạp     |
| Xử lý thanh toán                                   | Serverless (Lambda + API Gateway) | Bùng phát, hướng sự kiện, độc lập — lý tưởng cho mô hình tính phí theo lần gọi |

### 4. Kiến trúc giải pháp

Hệ thống theo mô hình hybrid với hai luồng xử lý riêng biệt, dùng chung một tầng dữ liệu:

**Backend cốt lõi (EC2 monolith)**  
Yêu cầu của người dùng được định tuyến qua Elastic Load Balancer đến nhóm EC2 Auto Scaling chạy ứng dụng FastAPI. Amazon Cognito xử lý xác thực, kiểm tra token trước khi request đến backend. Ứng dụng đọc và ghi dữ liệu đặt sân, người dùng và thông tin sân trực tiếp vào Amazon RDS (PostgreSQL), đồng thời lưu trữ ảnh sân và tài nguyên tĩnh trên Amazon S3.

**Luồng serverless (hướng sự kiện)**  
Thanh toán được xử lý qua Amazon API Gateway, kích hoạt một hàm AWS Lambda để thực hiện giao dịch. Sau khi xác nhận thanh toán, một hàm Lambda thứ hai ghi xác nhận đặt sân vào RDS và phát sự kiện lên Amazon SNS, gửi thông báo đến người dùng. Luồng này phù hợp với Lambda vì sự kiện thanh toán có tính bùng phát, độc lập và không cần tính toán liên tục.

**Tầng dữ liệu dùng chung**  
Cả monolith và các hàm serverless đều ghi vào cùng một cơ sở dữ liệu Amazon RDS (PostgreSQL). Cơ sở dữ liệu quan hệ được chọn thay vì DynamoDB vì đặt sân yêu cầu đảm bảo giao dịch - cụ thể là ngăn hai người dùng đặt cùng một khung giờ sân cùng lúc.

**Giám sát**  
Amazon CloudWatch thu thập log và số liệu từ cả EC2 và Lambda, cung cấp một giao diện thống nhất để khắc phục sự cố và theo dõi hiệu suất.

![Kiến trúc Hybrid Đặt sân (v3)](/images/2-Proposal/court_booking_hybrid_v3.png)

[Xem phiên bản SVG độ phân giải đầy đủ](/images/2-Proposal/court_booking_hybrid_v3.svg)

#### Diễn giải kiến trúc — 14 bước

**Frontend & xác thực (bước 1–3)**

| Bước | Luồng                       | Diễn giải                                                                                                                                        |
| ---- | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Người dùng → AWS Amplify    | **Visit app** — người dùng mở ứng dụng; frontend React + Vite được phục vụ toàn cầu từ CDN của AWS Amplify.                                          |
| 2    | Người dùng → Amazon Cognito | **Login** — người dùng đăng nhập (email/mật khẩu hoặc Google/Facebook) qua Cognito User Pool, nhận JWT ngắn hạn kèm refresh token.                   |
| 3    | Frontend Amplify → ELB      | **API calls** — frontend đã xác thực gọi các REST API backend (xác thực, sân, đặt sân, thanh toán) với JWT đính kèm dưới dạng Bearer token.          |

**Backend cốt lõi — EC2 monolith (bước 4–6)**

| Bước | Luồng             | Diễn giải                                                                                                                                                   |
| ---- | ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 4    | ELB → Amazon EC2  | **Forward request** — Elastic Load Balancer phân phối lưu lượng đến các instance FastAPI trong Auto Scaling Group; nhóm này tự co giãn theo tải.             |
| 5    | EC2 → Amazon RDS  | **Read / write booking data** — FastAPI đọc/ghi `users`, `courts`, `bookings`, `payments` trong PostgreSQL, áp dụng row-level lock + exclusion constraint để ngăn đặt trùng sân. |
| 6    | EC2 → Amazon S3   | **Read / write photos** — ảnh sân và tài nguyên tĩnh được lưu trữ và phục vụ từ S3.                                                                          |

**Luồng thanh toán serverless (bước 7–12)**

| Bước | Luồng                                   | Diễn giải                                                                                                                                  |
| ---- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 7    | Người dùng → Amazon API Gateway          | **Pay** — người dùng hoàn tất thanh toán với cổng thanh toán bên ngoài; webhook callback của cổng đổ về endpoint thanh toán trên API Gateway. |
| 8    | API Gateway → Lambda (Process payment)   | **Invoke** — API Gateway gọi Lambda xử lý thanh toán với payload của webhook.                                                                 |
| 9    | Lambda → Amazon RDS                      | **Write payment status** — Lambda xác thực webhook và cập nhật bản ghi `payments` (`SUCCESS`/`FAILED`, `transaction_id`, `gateway_response`). |
| 10   | Lambda → Lambda (Confirm booking)        | **Trigger** — khi thanh toán thành công, hàm xử lý kích hoạt Lambda xác nhận đặt sân.                                                         |
| 11   | Lambda → Amazon RDS                      | **Confirm slot** — Lambda xác nhận đặt `bookings.status = 'CONFIRMED'` (hoặc `CANCELLED` nếu thất bại), chốt khung giờ đã giữ.                |
| 12   | Lambda → Amazon SNS                      | **Publish** — Lambda xác nhận phát kết quả đặt sân lên topic thông báo SNS.                                                                   |

**Thông báo (bước 13–14)**

| Bước | Luồng             | Diễn giải                                                                                                     |
| ---- | ----------------- | -------------------------------------------------------------------------------------------------------------- |
| 13   | SNS → Amazon SES  | **Send email** — SNS chuyển tiếp đến SES để gửi email xác nhận đặt sân/thanh toán cho người dùng.               |
| 14   | SNS → Người dùng  | **Push notify** — đồng thời SNS đẩy thông báo trực tiếp về client của người dùng để phản hồi tức thì.           |

**Các luồng hỗ trợ (không đánh số):** developer đẩy code lên GitHub source repo (*Push code*), Amplify tự động **CI/CD deploy**; Amplify thực hiện **Auth check** với Cognito trước khi phục vụ các trang cần đăng nhập; Auto Scaling Group **scales** nhóm EC2; EC2 và cả hai Lambda **push logs** lên Amazon CloudWatch, nơi developer **view logs and metrics**.

### Dịch vụ AWS sử dụng

- **AWS Amplify**: Phục vụ frontend React + Vite qua CDN toàn cầu, với CI/CD từ GitHub source repo.
- **Amazon EC2 + Auto Scaling**: Chạy backend FastAPI; mở rộng ngang theo tải.
- **Elastic Load Balancer**: Phân phối lưu lượng người dùng đến các EC2 instance.
- **Amazon Cognito**: Quản lý danh tính người dùng và xác thực dựa trên token.
- **Amazon RDS (PostgreSQL)**: Nguồn dữ liệu duy nhất cho đặt sân, người dùng và sân.
- **Amazon S3**: Lưu trữ ảnh sân và tài nguyên tĩnh.
- **Amazon API Gateway**: Tiếp nhận yêu cầu thanh toán và chuyển đến Lambda.
- **AWS Lambda**: Xử lý thanh toán và ghi xác nhận đặt sân (hai hàm).
- **Amazon SNS**: Gửi thông báo xác nhận đặt sân đến người dùng.
- **Amazon SES**: Gửi email xác nhận đặt sân và thanh toán.
- **Amazon CloudWatch**: Logging và số liệu tập trung cho EC2 và Lambda.

### Thiết kế thành phần

- **Cân bằng tải**: ELB định tuyến lưu lượng HTTP đến nhóm EC2 Auto Scaling.
- **Tầng ứng dụng**: FastAPI trên EC2 xử lý logic duyệt, tìm kiếm và đặt sân.
- **Xác thực**: Cognito kiểm tra JWT trước khi request đến backend.
- **Tầng dữ liệu**: RDS đảm bảo tính toàn vẹn quan hệ và ngăn xung đột đặt sân đồng thời.
- **Lưu trữ tài nguyên**: S3 phục vụ ảnh sân và nội dung tĩnh.
- **Luồng thanh toán**: API Gateway → Lambda (thanh toán) → Lambda (ghi xác nhận vào RDS) → thông báo SNS.
- **Quan sát hệ thống**: CloudWatch tổng hợp log và số liệu từ cả EC2 và Lambda.

### 5. Lý do chọn kiến trúc Hybrid

Thiết kế thuần monolith sẽ buộc mọi khối lượng công việc - kể cả sự kiện thanh toán có tính bùng phát, độc lập - chạy qua cùng một EC2 fleet, dù không cần tính toán liên tục. Thiết kế thuần serverless sẽ gây ma sát đáng kể trong giai đoạn phát triển sớm (mô phỏng Lambda cold start và định tuyến API Gateway cục bộ rất phức tạp) - đây là ràng buộc quan trọng với timeline tám tuần của dự án.

Kiến trúc hybrid cho phép mỗi phần phát huy thế mạnh riêng. Logic đặt sân và tìm kiếm sân giữ nguyên trên EC2 vì đây là phần được truy cập thường xuyên nhất, nhạy cảm với độ trễ - server liên tục giúp tránh cold start và dễ phát triển, debug cục bộ hơn. Xử lý thanh toán và thông báo, ngược lại, có bản chất hướng sự kiện và ít xảy ra hơn so với lưu lượng duyệt sân, nên Lambda và SNS là lựa chọn tự nhiên: nhóm chỉ trả tiền cho tính toán khi có sự kiện thanh toán thực sự xảy ra.

Cách tiếp cận này cũng cho phép nhóm thể hiện năng lực với cả hai mô hình triển khai trong cùng một dự án - phản ánh trực tiếp mục tiêu tuần 3 của chương trình AWS FCJ về triển khai với EC2, Lambda và API Gateway - mà không làm phức tạp thêm kiến trúc ứng dụng cốt lõi.

### 6. Thiết kế API & Cơ sở dữ liệu thống nhất

Phần này hợp nhất ba bản thiết kế backend riêng lẻ — thiết kế thanh toán của Hiếu (Tuần 2), thiết kế đặt sân của Thành và thiết kế xác thực của Nguyên (Tuần 3) — thành một đặc tả chuẩn duy nhất, theo phân công tại buổi họp nhóm ngày 05/07/2026.

**Nguyên tắc chủ đạo: tối đa hóa việc sử dụng các dịch vụ được quản lý của AWS.** Bất cứ khi nào một dịch vụ AWS đã đáp ứng được một nhu cầu, thiết kế sẽ giao phó cho dịch vụ đó thay vì tự xây dựng lại trong cơ sở dữ liệu hoặc mã ứng dụng. Hệ quả lớn nhất nằm ở phần danh tính: Amazon Cognito là nơi duy nhất quản lý thông tin đăng nhập và phiên làm việc, giúp schema quan hệ chỉ còn **5 bảng**.

| Nhu cầu                                   | Do dịch vụ nào đảm nhận                              | Thay thế cho (từ các bản thiết kế riêng)   |
| ------------------------------------------ | ----------------------------------------------------- | ------------------------------------------- |
| Lưu trữ thông tin đăng nhập (mật khẩu)     | Cognito User Pool                                     | Cột `users.password_hash`                   |
| Đăng nhập mạng xã hội (Google / Facebook)  | Cognito federated identity providers (Hosted UI)      | Bảng `user_identities`                      |
| Vòng đời phiên, refresh, thu hồi           | Cognito refresh token + `GlobalSignOut`               | Bảng `refresh_tokens`                       |
| Vai trò & phân quyền (RBAC)                | Cognito Groups (claim `cognito:groups` trong JWT)     | Bảng `roles` + bảng cầu nối `user_roles`    |
| Người dùng đa vai trò (player + court_owner) | Một người dùng có thể thuộc nhiều Cognito Group     | Thiết kế many-to-many `user_roles`          |

#### 6.1 Tài liệu API thống nhất (15 endpoint)

**Xác thực — 5 endpoint** (giữ nguyên hợp đồng API từ thiết kế của Nguyên, triển khai lại dưới dạng FastAPI wrapper quanh Cognito)

| # | Endpoint                | Method | Input                                     | Output                                             | Use Case                                                                                                                          |
| - | ----------------------- | ------ | ------------------------------------------ | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| 1 | `/api/v1/auth/register` | `POST` | `email`, `password`, `full_name`           | `user_id`, `message: "Success"`                     | Đăng ký tài khoản mới qua Cognito `SignUp`; sau khi xác nhận, chèn bản ghi hồ sơ vào `users` liên kết qua `cognito_sub`.               |
| 2 | `/api/v1/auth/login`    | `POST` | `email`, `password`                        | `access_token` (JWT), `refresh_token`, `user_data`  | Đăng nhập chuẩn qua Cognito `InitiateAuth`. Cognito cấp JWT ngắn hạn và refresh token có thể thu hồi — không cần bảng phiên.           |
| 3 | `/api/v1/auth/oauth`    | `POST` | `provider` (google/facebook), `auth_code`  | `access_token` (JWT), `refresh_token`, `user_data`  | OAuth qua Cognito federated identity providers. Lần đăng nhập mạng xã hội đầu tiên tự động upsert bản ghi hồ sơ `users`.               |
| 4 | `/api/v1/auth/refresh`  | `POST` | `refresh_token`                            | `access_token` (JWT mới)                            | Đổi refresh token Cognito lấy access token mới (luồng `REFRESH_TOKEN_AUTH`) khi JWT ~15 phút hết hạn.                                  |
| 5 | `/api/v1/auth/logout`   | `POST` | Bearer token (header)                      | `204 No Content`                                    | Gọi Cognito `GlobalSignOut`, thu hồi toàn bộ refresh token của người dùng — đăng xuất từ xa tức thì mà không cần ghi DB.               |

**Đặt sân — 6 endpoint** (từ thiết kế của Thành, đã hoàn thiện đặc tả input/output)

| # | Endpoint                                  | Method   | Input                                                                | Output                                                       | Use Case                                                                                                            |
| - | ----------------------------------------- | -------- | --------------------------------------------------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 1 | `/api/v1/courts`                          | `GET`    | `sport_type`, `address`, `page`, `limit` (query)                       | `{ data: [...courts], total, page, limit }`                    | Người dùng tìm kiếm và duyệt các sân đang hoạt động theo địa điểm hoặc loại hình thể thao.                              |
| 2 | `/api/v1/courts/{court_id}/availability`  | `GET`    | `court_id` (path), `date` (query)                                      | `{ court_id, date, booked_slots: [{start_time, end_time}] }`   | Lấy các khung giờ đã được đặt của sân theo ngày, để frontend vô hiệu hóa các nút khung giờ tương ứng.                   |
| 3 | `/api/v1/bookings`                        | `POST`   | Bearer token; `court_id`, `start_time`, `end_time`, `note` (tùy chọn)  | `booking_id`, `total_amount`, `status: "PENDING"` — hoặc `409 Conflict` | Tạo yêu cầu đặt sân tạm thời (PENDING) khi checkout. Áp dụng row-level locking để ngăn đặt trùng sân.          |
| 4 | `/api/v1/bookings/{booking_id}`           | `PUT`    | Bearer token; `start_time`, `end_time`                                 | Bản ghi booking đã cập nhật                                    | Người dùng đổi khung giờ của yêu cầu đặt sân chưa thanh toán (chỉ trạng thái PENDING). Chạy lại kiểm tra chồng lấp.     |
| 5 | `/api/v1/bookings/{booking_id}`           | `DELETE` | Bearer token; `booking_id` (path)                                      | `204 No Content` (`status → CANCELLED`)                        | Hủy đặt sân (chưa thanh toán, hoặc đã xác nhận theo chính sách hủy của sân).                                            |
| 6 | `/api/v1/bookings/me`                     | `GET`    | Bearer token; `status`, `page`, `limit` (query)                        | `{ data: [...bookings], total, page, limit }`                  | Lấy danh sách đặt sân hiện tại và lịch sử của người dùng đã xác thực (`user_id` trích từ JWT).                          |

**Thanh toán — 4 endpoint** (từ thiết kế Tuần 2 của Hiếu, giữ nguyên)

| # | Endpoint                         | Method | Input                                                    | Output                                            | Use Case                                                                                                              |
| - | -------------------------------- | ------ | ---------------------------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| 1 | `/api/v1/payments`               | `POST` | `booking_id`, `amount`, `payment_method`                   | `payment_id`, `checkout_url`, `status: "PENDING"`   | Người dùng khởi tạo thanh toán sau khi xác nhận đặt sân; tạo bản ghi PENDING và trả về URL thanh toán.                    |
| 2 | `/api/v1/payments/webhook`       | `POST` | `payment_id`, `transaction_id`, `status`, `gateway_data`   | `{ success: boolean }`                              | Callback từ cổng thanh toán — điểm vào của luồng serverless: API Gateway → Lambda cập nhật RDS → SNS thông báo người dùng. |
| 3 | `/api/v1/payments/{payment_id}`  | `GET`  | `payment_id` (path), Bearer token                          | Bản ghi thanh toán đầy đủ                           | Người dùng hoặc admin kiểm tra trạng thái của một giao dịch cụ thể.                                                       |
| 4 | `/api/v1/payments`               | `GET`  | Bearer token; `page`, `limit`                              | `{ data: [...payments], total, page, limit }`       | Người dùng xem lịch sử thanh toán với phân trang.                                                                          |

#### 6.2 Thiết kế Cơ sở dữ liệu thống nhất (5 bảng)

Bảng duy nhất thay đổi về cấu trúc là `users` — được tái cấu trúc xoay quanh Cognito như nguồn quản lý danh tính:

**`users`** (đã điều chỉnh)

| Cột           | Kiểu dữ liệu   | Ràng buộc            | Mô tả                                                                                    |
| ------------- | -------------- | -------------------- | ----------------------------------------------------------------------------------------- |
| `id`          | `UUID`         | `PRIMARY KEY`        | Mã định danh người dùng                                                                   |
| `cognito_sub` | `VARCHAR(255)` | `UNIQUE, NOT NULL`   | Subject ID từ Cognito User Pool — liên kết duy nhất giữa JWT và bản ghi hồ sơ cục bộ      |
| `email`       | `VARCHAR(255)` | `UNIQUE, NOT NULL`   | Email đăng nhập (đồng bộ từ Cognito)                                                      |
| `full_name`   | `VARCHAR(255)` | `NOT NULL`           | Tên hiển thị                                                                              |
| `phone`       | `VARCHAR(20)`  |                      | Số điện thoại liên hệ                                                                     |
| `avatar_url`  | `VARCHAR(500)` |                      | URL ảnh đại diện trên S3                                                                  |
| `role`        | `VARCHAR(20)`  | `DEFAULT 'player'`   | Vai trò chính, cache để truy vấn DB; việc phân quyền dùng claim `cognito:groups` trong JWT |
| `is_active`   | `BOOLEAN`      | `DEFAULT true`       | Cờ xóa mềm / tạm khóa tài khoản                                                           |
| `created_at`  | `TIMESTAMP`    | `DEFAULT NOW()`      | Thời điểm tạo tài khoản                                                                   |
| `updated_at`  | `TIMESTAMP`    | `DEFAULT NOW()`      | Lần cập nhật hồ sơ gần nhất                                                               |

> `password_hash` được cố ý loại bỏ: thông tin đăng nhập chỉ tồn tại trong Cognito. Các bảng `courts`, `court_images`, `bookings` và `payments` giữ nguyên — đặc tả cột đầy đủ xem tại [worklog Tuần 2](/vi/1-worklog/1.2-week2/).

{{<mermaid>}}
erDiagram
users {
UUID id PK
VARCHAR cognito_sub UK
VARCHAR email UK
VARCHAR full_name
VARCHAR phone
VARCHAR avatar_url
VARCHAR role
BOOLEAN is_active
TIMESTAMP created_at
TIMESTAMP updated_at
}
courts {
UUID id PK
UUID owner_id FK
VARCHAR name
TEXT description
VARCHAR address
VARCHAR sport_type
DECIMAL price_per_hour
VARCHAR status
TIMESTAMP created_at
TIMESTAMP updated_at
}
court_images {
UUID id PK
UUID court_id FK
VARCHAR image_url
BOOLEAN is_primary
TIMESTAMP created_at
}
bookings {
UUID id PK
UUID user_id FK
UUID court_id FK
TIMESTAMP start_time
TIMESTAMP end_time
DECIMAL total_amount
VARCHAR status
TEXT note
TIMESTAMP created_at
TIMESTAMP updated_at
}
payments {
UUID id PK
UUID booking_id FK
UUID user_id FK
DECIMAL amount
VARCHAR currency
VARCHAR status
VARCHAR payment_method
VARCHAR transaction_id UK
JSONB gateway_response
TIMESTAMP created_at
TIMESTAMP updated_at
}

    users ||--o{ courts : "owns"
    users ||--o{ bookings : "makes"
    users ||--o{ payments : "pays"
    courts ||--o{ court_images : "has"
    courts ||--o{ bookings : "reserved via"
    bookings ||--o| payments : "paid by"

{{</mermaid>}}

#### 6.3 Cơ chế ngăn đặt trùng sân (áp dụng từ thiết kế của Thành)

Cơ chế phòng thủ hai lớp ở tầng cơ sở dữ liệu, thay thế cho ràng buộc không điều kiện ban đầu:

**Lớp 1 — Row-level locking trong transaction đặt sân.** `POST /api/v1/bookings` khóa bản ghi sân cha (`SELECT ... FOR UPDATE`), kiểm tra các đặt sân chồng lấp chưa bị hủy, sau đó chèn với trạng thái `PENDING` — trả về `409 Conflict` và rollback nếu phát hiện chồng lấp.

**Lớp 2 — Partial exclusion constraint** là tuyến phòng thủ cuối cùng, kể cả với SQL thủ công:

```sql
ALTER TABLE bookings
ADD CONSTRAINT chk_bookings_overlap
EXCLUDE USING gist (
  court_id WITH =,
  tsrange(start_time, end_time) WITH &&
)
WHERE (status != 'CANCELLED');
```

#### 6.4 Khác biệt so với các phiên bản riêng lẻ

**So với phiên bản của Hiếu (Tuần 2):**

| Khía cạnh              | Phiên bản của Hiếu                        | Phiên bản thống nhất                                                            |
| ---------------------- | ------------------------------------------ | -------------------------------------------------------------------------------- |
| `users.password_hash`  | `NOT NULL` (bcrypt)                        | **Loại bỏ** — thông tin đăng nhập chỉ tồn tại trong Cognito                       |
| `users.cognito_sub`    | `UNIQUE`, cho phép null                    | **`UNIQUE, NOT NULL`** — liên kết danh tính bắt buộc                              |
| `users.is_active`      | Không có                                   | **Bổ sung** (từ Nguyên) — xóa mềm / tạm khóa                                      |
| Exclusion constraint   | Không điều kiện trên `(court_id, tsrange)` | **Tinh chỉnh** với `WHERE (status != 'CANCELLED')` + kết hợp row-level locking    |
| Phạm vi API            | 4 endpoint thanh toán                      | **15 endpoint** trải khắp xác thực, đặt sân và thanh toán                         |
| Thiết kế thanh toán    | —                                          | Giữ nguyên **không đổi** (endpoint, bảng `payments`, luồng 9 bước)                |

**So với phiên bản của Nguyên:**

| Khía cạnh                  | Phiên bản của Nguyên                                   | Phiên bản thống nhất                                                                                      |
| -------------------------- | ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Quy mô schema              | 9 bảng                                                  | **5 bảng** — 4 bảng danh tính được thay bằng Cognito                                                         |
| OAuth (Google/Facebook)    | Bảng `user_identities`                                  | **Cognito federated identity providers** (Hosted UI)                                                         |
| Phiên & buộc đăng xuất     | Bảng `refresh_tokens` với `is_revoked`                  | **Cognito refresh token + `GlobalSignOut`** — cùng khả năng, không cần ghi DB                                |
| RBAC / đa vai trò          | Bảng `roles` + `user_roles`                             | **Cognito Groups** — người dùng thuộc nhiều group vẫn giữ được yêu cầu đa vai trò; JWT mang `cognito:groups` |
| `users.username`           | Cột tùy chọn                                            | Loại bỏ — dùng thuộc tính `preferred_username` của Cognito                                                   |
| Hợp đồng API xác thực      | 5 endpoint thao tác trên bảng cục bộ                    | **Giữ nguyên 5 hợp đồng**, triển khai lại dưới dạng FastAPI wrapper quanh Cognito API                        |
| Luồng xác thực             | JWT ngắn hạn + refresh token tự quản lý                 | Luồng giống hệt, với **Cognito là bên cấp phát token**                                                       |

**So với phiên bản của Thành:**

| Khía cạnh                  | Phiên bản của Thành                                       | Phiên bản thống nhất                                                    |
| -------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------- |
| Endpoint đặt sân           | 6 endpoint, input/output chưa đặc tả                       | **Giữ nguyên cả 6; đã hoàn thiện đặc tả input/output**                     |
| Cơ chế ngăn đặt trùng      | Thiết kế hai lớp (row lock + partial constraint)           | **Áp dụng nguyên vẹn làm cơ chế chuẩn**                                    |
| Bảng `users` tham chiếu    | Schema Tuần 2 gốc (`password_hash`, `cognito_sub` null)    | Cập nhật theo thiết kế `users` thống nhất xoay quanh Cognito               |
| Luồng thanh toán & vòng đời | Kế thừa từ thiết kế Tuần 2 của Hiếu                        | Không đổi                                                                  |

### 7. Lộ trình & Mốc triển khai

| Tuần | Công việc                                                     |
| ---- | ------------------------------------------------------------- |
| 1–2  | Thiết lập VPC, EC2, RDS, Cognito; triển khai skeleton FastAPI |
| 3    | Cấu hình Auto Scaling và ELB; tích hợp xác thực Cognito       |
| 4    | Xây dựng logic đặt sân; đảm bảo ràng buộc giao dịch trên RDS  |
| 5    | Triển khai luồng thanh toán API Gateway + Lambda              |
| 6    | Kết nối thông báo SNS; Lambda xác nhận ghi vào RDS            |
| 7    | Thiết lập CloudWatch dashboard và cảnh báo; tích hợp S3       |
| 8    | Kiểm thử đầu cuối, tinh chỉnh hiệu suất, hoàn thiện tài liệu  |

### 8. Ước tính ngân sách

| Dịch vụ                               | Chi phí ước tính/tháng |
| ------------------------------------- | ---------------------- |
| Amazon EC2 (t3.small × 2)             | ~$15.00                |
| Amazon RDS (db.t3.micro, Single-AZ)   | ~$13.00                |
| Elastic Load Balancer                 | ~$16.00                |
| Amazon Cognito (≤50.000 MAU miễn phí) | $0.00                  |
| AWS Lambda (tính phí theo lần gọi)    | ~$0.00–$1.00           |
| Amazon API Gateway                    | ~$0.01                 |
| Amazon SNS                            | ~$0.00                 |
| Amazon S3                             | ~$0.50                 |
| Amazon CloudWatch                     | ~$1.00                 |
| **Tổng**                              | **~$45–$47/tháng**     |

Chi phí có thể giảm đáng kể bằng cách sử dụng EC2 Reserved Instances hoặc Savings Plans cho môi trường production.

### 9. Đánh giá rủi ro

#### Ma trận rủi ro

- **Race condition đặt trùng sân**: Ảnh hưởng cao, xác suất thấp - giảm thiểu bằng giao dịch RDS và row-level locking.
- **EC2 instance lỗi**: Ảnh hưởng cao, xác suất thấp - giảm thiểu bằng Auto Scaling và health check của ELB.
- **Lambda cold start khi thanh toán**: Ảnh hưởng trung bình, xác suất thấp - giảm thiểu bằng provisioned concurrency nếu cần.
- **Vượt ngân sách**: Ảnh hưởng trung bình, xác suất thấp - giảm thiểu bằng cảnh báo billing CloudWatch và chính sách cooldown Auto Scaling.

#### Chiến lược giảm thiểu

- Sử dụng `SELECT FOR UPDATE` hoặc advisory lock của PostgreSQL để ngăn xung đột đặt sân đồng thời ở tầng cơ sở dữ liệu.
- Cấu hình health check ELB để tự động thay thế EC2 instance không hoạt động.
- Đặt cảnh báo AWS Budget để thông báo khi chi tiêu hàng tháng vượt ngưỡng định sẵn.

### 10. Kết quả kỳ vọng

- **Kỹ thuật**: Hệ thống đặt sân hoàn chỉnh với thông tin thời gian thực, thanh toán bảo mật và thông báo tức thì - triển khai trên kiến trúc AWS hybrid cấp độ production.
- **Học thuật**: Thể hiện kinh nghiệm thực hành với EC2, RDS, Cognito, Lambda, API Gateway, SNS, S3 và CloudWatch trong một dự án duy nhất.
- **Vận hành**: Auto Scaling đảm bảo hệ thống xử lý tốt các đợt tăng lưu lượng mà không cần can thiệp thủ công; CloudWatch cung cấp khả năng quan sát liên tục cho sức khỏe hệ thống.
