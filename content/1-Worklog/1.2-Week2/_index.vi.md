---
title: "Worklog Tuần 2"
date: 2026-06-22
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Mục tiêu tuần 2:

- Học sâu các dịch vụ AWS trên AWS Skill Builder để chuẩn bị cho chứng chỉ AWS Developer Associate.
- Hoàn thiện tài liệu API thanh toán và thiết kế cơ sở dữ liệu đầy đủ cho ứng dụng đặt sân.
- Trình bày logic thanh toán và thiết kế DB cho nhóm; xem xét bản UI draft.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc                                                                                                                                                                                                                                                        | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                                                                        |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ------------------------------------------------------------------------------------- |
| 2   | - **AWS Skill Builder — Bộ câu hỏi thực hành chính thức: AWS Certified Developer – Associate (DVA-C02)** <br> - Xem lại kiến trúc ứng dụng đặt sân; lên kế hoạch các kết quả bàn giao trong tuần                                                                 | 22/06/2026   | 22/06/2026      | [AWS Skill Builder](https://skillsprofile.skillbuilder.aws/user/minervaph/cloudquest) |
| 4   | - **AWS Skill Builder — Cloud Quest: Cloud Practitioner** <br>&emsp; + Cloud Computing Essentials <br>&emsp; + Cloud First Steps <br>&emsp; + Computing Solutions (AI mode) <br> - **AWS Escape Room**: Exam Prep for AWS Certified Cloud Practitioner (CLF-C02) | 24/06/2026   | 24/06/2026      | [AWS Skill Builder](https://skillsprofile.skillbuilder.aws/user/minervaph/cloudquest) |
| 5   | - Hoàn thiện tài liệu API thanh toán (4 endpoint) <br> - Thiết kế schema bảng `payments`                                                                                                                                                                         | 25/06/2026   | 25/06/2026      |                                                                                       |
| 6   | - **AWS Skill Builder — Cloud Quest: Cloud Practitioner** <br>&emsp; + Networking Concepts <br>&emsp; + Computing Solutions <br> - Hoàn thiện thiết kế DB đầy đủ (5 bảng + ERD) <br> - Mô tả luồng thanh toán đầu cuối                                           | 26/06/2026   | 26/06/2026      | [AWS Skill Builder](https://skillsprofile.skillbuilder.aws/user/minervaph/cloudquest) |
| 7   | - **Họp nhóm (27/06/2026)**: trình bày logic thanh toán & thiết kế DB <br> - Xem xét bản UI draft của Danh                                                                                                                                                       | 27/06/2026   | 27/06/2026      |                                                                                       |

### Kết quả đạt được tuần 2:

**AWS Skill Builder — Bộ câu hỏi thực hành chính thức: AWS Certified Developer – Associate (DVA-C02)**

Hoàn thành vào **22/06/2026** — bộ câu hỏi luyện thi chính thức dạng thi thật cho chứng chỉ DVA-C02, bao gồm các chủ đề AWS Developer Associate cốt lõi: Lambda, API Gateway, DynamoDB, SQS, SNS, Cognito và các dịch vụ triển khai.

[Tải chứng chỉ hoàn thành](/images/1-Worklog/1.2-Week2/CompletionCert-OfficialPracticeQuestionSet-DVA-C02-20260622.pdf)

---

**AWS Skill Builder — Cloud Quest: Cloud Practitioner**

| Solution                   | Hoàn thành                        | Mô tả                                                                                                                            |
| -------------------------- | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Cloud Computing Essentials | 24/06/2026                        | Định nghĩa điện toán đám mây, so sánh với hạ tầng truyền thống và đề xuất giá trị. Lab: bật static website hosting trên S3.      |
| Cloud First Steps          | 24/06/2026                        | Hạ tầng toàn cầu AWS — Regions, Availability Zones, data centers và Points of Presence. Lab: triển khai EC2 trên hai AZ.         |
| Computing Solutions        | 24/06/2026 (AI mode) / 26/06/2026 | Tổng quan về các dịch vụ compute của AWS. Lab: lọc loại EC2 instance, kết nối qua Session Manager, thay đổi kích thước instance. |
| Networking Concepts        | 26/06/2026                        | Tổng quan về mạng trên AWS — VPC, subnet, route table, internet gateway và security group. Lab: cấu hình VPC cho tầng web + DB.  |

Điểm danh tiếng đạt được:

| Dịch vụ    | Điểm |
| ---------- | ---- |
| Amazon EC2 | 34   |
| Amazon VPC | 10   |
| AWS IAM    | 1    |
| Amazon S3  | 14   |

**Minh chứng hoàn thành**

_Cloud Computing Essentials — 24/06/2026_

![Minh chứng hoàn thành Cloud Computing Essentials](/images/1-Worklog/1.2-Week2/CloudComputingEssentials_20260624092947.png)

_Cloud First Steps — 24/06/2026_

![Minh chứng hoàn thành Cloud First Steps](/images/1-Worklog/1.2-Week2/CloudFirstSteps_20260624103147.png)

_Computing Solutions — 24/06/2026 (AI mode)_

![Minh chứng hoàn thành Computing Solutions AI mode](/images/1-Worklog/1.2-Week2/ComputingSolutions_AI-mode_20260624154735.png)

_Computing Solutions — 26/06/2026_

![Minh chứng hoàn thành Computing Solutions](/images/1-Worklog/1.2-Week2/ComputingSolutions_20260626163021.png)

_Networking Concepts — 26/06/2026 (AI mode)_

![Minh chứng hoàn thành Networking Concepts AI mode](/images/1-Worklog/1.2-Week2/NetworkingConcepts_AI-mode_20260626154612.png)

_Networking Concepts — 26/06/2026_

![Minh chứng hoàn thành Networking Concepts](/images/1-Worklog/1.2-Week2/NetworkingConcepts_20260626161508.png)

**AWS Escape Room: Exam Prep for AWS Certified Cloud Practitioner (CLF-C02)**

Hoàn thành chế độ luyện tập đơn người chơi vào **24/06/2026** — phòng thoát hiểm 3D ảo kéo dài 2–3 giờ với các câu hỏi dạng thi và bài tập thực hành bao gồm: Amazon API Gateway, Amazon DynamoDB, Amazon EC2, AWS IAM, AWS Lambda và Amazon S3.

[Tải chứng chỉ hoàn thành](/images/1-Worklog/1.2-Week2/CompletionCert-EscapeRoom.pdf)

**Kết quả tính năng Thanh toán**

- Thiết kế và tài liệu hóa 4 endpoint API thanh toán (khởi tạo, webhook, lấy theo ID, lịch sử).
- Thiết kế schema bảng `payments` với 11 cột bao gồm định danh, quan hệ, vòng đời trạng thái và kiểm tra cổng thanh toán.
- Hoàn thiện thiết kế cơ sở dữ liệu đầy đủ 5 bảng (`users`, `courts`, `court_images`, `bookings`, `payments`) cùng ERD.
- Tài liệu hóa luồng thanh toán đầu cuối 9 bước, ánh xạ từng hành động với thao tác DB và dịch vụ AWS.

---

### Biên bản họp nhóm — 27/06/2026

**Thành phần tham dự:** Hiếu, Thành, Nguyên, Hùng
**Vắng mặt:** Danh

**Trình bày**

- **Hiếu** trình bày logic thanh toán đầu cuối và hướng dẫn nhóm qua thiết kế cơ sở dữ liệu đề xuất cho ứng dụng.
- **Danh** (gửi trước qua kênh nhóm) chia sẻ bản UI draft ban đầu để cả nhóm xem xét.

**Phân công công việc**

| Nhóm                          | Người phụ trách | Kết quả bàn giao                                                                              |
| ----------------------------- | --------------- | --------------------------------------------------------------------------------------------- |
| Frontend — Thiết kế UI        | Danh & Hùng     | Thống nhất và cải thiện giao diện dựa trên bản draft của Danh                                 |
| Frontend — Tech Stack         | Danh & Hùng     | Đề xuất tech stack cho phần frontend                                                          |
| Backend — Tech Stack          | Tất cả BE       | Đề xuất tech stack cho phần backend                                                           |
| Backend — Kiến trúc           | Hiếu            | Tích hợp Amazon Amplify vào kiến trúc để phục vụ frontend; vẽ lại sơ đồ kiến trúc cập nhật    |
| Backend — Quản lý đặt sân     | Thành           | Tài liệu API (tên endpoint, input, output, use case) và thiết kế DB (bảng, cột, kiểu dữ liệu) |
| Backend — Xác thực người dùng | Nguyên          | Tài liệu API (tên endpoint, input, output, use case) và thiết kế DB (bảng, cột, kiểu dữ liệu) |

---

### Kết quả của Hieu — Tính năng Thanh toán

#### Tài liệu API

<!--
| # | Endpoint | Method | Input | Output | Use Case |
|---|----------|--------|-------|--------|----------|
| 1 | `/api/v1/payments` | `POST` | `booking_id` (UUID), `amount` (decimal), `payment_method` (string) | `payment_id` (UUID), `checkout_url` (string), `status: "PENDING"` | Người dùng khởi tạo thanh toán sau khi xác nhận đặt sân. Tạo bản ghi thanh toán trạng thái PENDING trong RDS và trả về URL thanh toán để chuyển hướng người dùng. |
| 2 | `/api/v1/payments/webhook` | `POST` | `payment_id` (UUID), `transaction_id` (string), `status` (string), `gateway_data` (object) | `{ success: boolean }` | Cổng thanh toán thông báo kết quả giao dịch cho hệ thống. Đây là điểm vào của luồng serverless: API Gateway nhận callback → gọi Lambda cập nhật trạng thái thanh toán trong RDS → kích hoạt SNS thông báo cho người dùng. |
| 3 | `/api/v1/payments/{payment_id}` | `GET` | `payment_id` (path param), Bearer token (header) | `payment_id`, `booking_id`, `amount`, `currency`, `status`, `payment_method`, `transaction_id`, `created_at`, `updated_at` | Người dùng hoặc admin kiểm tra trạng thái của một giao dịch cụ thể. |
| 4 | `/api/v1/payments` | `GET` | Bearer token (header), `page` (int), `limit` (int) | `{ data: [...payments], total, page, limit }` | Người dùng xem lịch sử thanh toán với phân trang. |
-->

<table style="width:100%; table-layout:fixed; word-break:break-word;">
  <colgroup>
    <col style="width:3%">
    <col style="width:15%">
    <col style="width:5%">
    <col style="width:20%">
    <col style="width:15%">
    <col style="width:42%">
  </colgroup>
  <thead>
    <tr>
      <th>#</th>
      <th>Endpoint</th>
      <th>Method</th>
      <th>Input</th>
      <th>Output</th>
      <th>Use Case</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td><code>/api/v1/payments</code></td>
      <td><code>POST</code></td>
      <td><code>booking_id</code> (UUID), <code>amount</code> (decimal), <code>payment_method</code> (string)</td>
      <td><code>payment_id</code> (UUID), <code>checkout_url</code> (string), <code>status: "PENDING"</code></td>
      <td>Người dùng khởi tạo thanh toán sau khi xác nhận đặt sân. Tạo bản ghi thanh toán trạng thái PENDING trong RDS và trả về URL thanh toán để chuyển hướng người dùng.</td>
    </tr>
    <tr>
      <td>2</td>
      <td><code>/api/v1/payments/webhook</code></td>
      <td><code>POST</code></td>
      <td><code>payment_id</code> (UUID), <code>transaction_id</code> (string), <code>status</code> (string), <code>gateway_data</code> (object)</td>
      <td><code>{ success: boolean }</code></td>
      <td>Cổng thanh toán thông báo kết quả giao dịch cho hệ thống. Đây là điểm vào của luồng serverless: API Gateway nhận callback → gọi Lambda cập nhật trạng thái thanh toán trong RDS → kích hoạt SNS thông báo cho người dùng.</td>
    </tr>
    <tr>
      <td>3</td>
      <td><code>/api/v1/payments/{payment_id}</code></td>
      <td><code>GET</code></td>
      <td><code>payment_id</code> (path param), Bearer token (header)</td>
      <td><code>payment_id</code>, <code>booking_id</code>, <code>amount</code>, <code>currency</code>, <code>status</code>, <code>payment_method</code>, <code>transaction_id</code>, <code>created_at</code>, <code>updated_at</code></td>
      <td>Người dùng hoặc admin kiểm tra trạng thái của một giao dịch cụ thể.</td>
    </tr>
    <tr>
      <td>4</td>
      <td><code>/api/v1/payments</code></td>
      <td><code>GET</code></td>
      <td>Bearer token (header), <code>page</code> (int), <code>limit</code> (int)</td>
      <td><code>{ data: [...payments], total, page, limit }</code></td>
      <td>Người dùng xem lịch sử thanh toán với phân trang.</td>
    </tr>
  </tbody>
</table>

#### Thiết kế Cơ sở dữ liệu

**Bảng: `payments`**

| Cột                | Kiểu dữ liệu     | Ràng buộc                       | Mô tả                                                                   |
| ------------------ | ---------------- | ------------------------------- | ----------------------------------------------------------------------- |
| `id`               | `UUID`           | `PRIMARY KEY`                   | Mã định danh thanh toán duy nhất                                        |
| `booking_id`       | `UUID`           | `NOT NULL`, `FK → bookings(id)` | Đặt sân tương ứng với thanh toán này                                    |
| `user_id`          | `UUID`           | `NOT NULL`, `FK → users(id)`    | Người dùng khởi tạo thanh toán                                          |
| `amount`           | `DECIMAL(12, 2)` | `NOT NULL`                      | Số tiền thanh toán                                                      |
| `currency`         | `VARCHAR(3)`     | `NOT NULL`, `DEFAULT 'VND'`     | Mã tiền tệ                                                              |
| `status`           | `VARCHAR(20)`    | `NOT NULL`, `DEFAULT 'PENDING'` | `PENDING` / `SUCCESS` / `FAILED` / `REFUNDED`                           |
| `payment_method`   | `VARCHAR(50)`    | `NOT NULL`                      | `CREDIT_CARD` / `BANK_TRANSFER` / `E_WALLET`                            |
| `transaction_id`   | `VARCHAR(255)`   | `UNIQUE`                        | Mã giao dịch từ cổng thanh toán                                         |
| `gateway_response` | `JSONB`          |                                 | Dữ liệu phản hồi thô từ cổng thanh toán để kiểm tra và xử lý tranh chấp |
| `created_at`       | `TIMESTAMP`      | `NOT NULL`, `DEFAULT NOW()`     | Thời điểm khởi tạo thanh toán                                           |
| `updated_at`       | `TIMESTAMP`      | `NOT NULL`, `DEFAULT NOW()`     | Thời điểm cập nhật trạng thái gần nhất                                  |

**Ghi chú:**

- `transaction_id` là nullable cho đến khi cổng thanh toán phản hồi (giữ giá trị `NULL` khi trạng thái là `PENDING`).
- `gateway_response` lưu toàn bộ payload thô từ cổng thanh toán — hữu ích cho việc xử lý tranh chấp và debug mà không cần truy vấn lại cổng thanh toán.
- Cột `status` được cập nhật bởi Lambda xác nhận (luồng webhook), không phải bởi EC2 monolith — giữ rõ ràng sự tách biệt giữa hai luồng xử lý.

---

### Thiết kế Cơ sở dữ liệu đầy đủ

#### Các bảng

**`users`**

| Cột             | Kiểu dữ liệu   | Ràng buộc          | Mô tả                                                                        |
| --------------- | -------------- | ------------------ | ---------------------------------------------------------------------------- |
| `id`            | `UUID`         | `PRIMARY KEY`      | Mã định danh người dùng                                                      |
| `email`         | `VARCHAR(255)` | `UNIQUE, NOT NULL` | Email đăng nhập                                                              |
| `password_hash` | `VARCHAR(255)` | `NOT NULL`         | Mật khẩu đã mã hóa (Bcrypt)                                                  |
| `full_name`     | `VARCHAR(255)` | `NOT NULL`         | Tên hiển thị                                                                 |
| `phone`         | `VARCHAR(20)`  |                    | Số điện thoại liên hệ                                                        |
| `cognito_sub`   | `VARCHAR(255)` | `UNIQUE`           | Subject ID từ Cognito User Pool — liên kết JWT với bản ghi người dùng cục bộ |
| `role`          | `VARCHAR(20)`  | `DEFAULT 'player'` | `player` / `admin` / `court_owner`                                           |
| `avatar_url`    | `VARCHAR(500)` |                    | URL ảnh đại diện trên S3                                                     |
| `created_at`    | `TIMESTAMP`    | `DEFAULT NOW()`    | Thời điểm tạo tài khoản                                                      |
| `updated_at`    | `TIMESTAMP`    | `DEFAULT NOW()`    | Lần cập nhật hồ sơ gần nhất                                                  |

**`courts`**

| Cột              | Kiểu dữ liệu    | Ràng buộc          | Mô tả                                              |
| ---------------- | --------------- | ------------------ | -------------------------------------------------- |
| `id`             | `UUID`          | `PRIMARY KEY`      | Mã định danh sân                                   |
| `owner_id`       | `UUID`          | `FK → users(id)`   | Chủ sân (người dùng có role `court_owner`)         |
| `name`           | `VARCHAR(255)`  | `NOT NULL`         | Tên sân                                            |
| `description`    | `TEXT`          |                    | Mô tả sân                                          |
| `address`        | `VARCHAR(500)`  | `NOT NULL`         | Địa chỉ thực tế                                    |
| `sport_type`     | `VARCHAR(50)`   | `NOT NULL`         | `badminton` / `tennis` / `football` / `basketball` |
| `price_per_hour` | `DECIMAL(12,2)` | `NOT NULL`         | Giá thuê theo giờ (VND)                            |
| `status`         | `VARCHAR(20)`   | `DEFAULT 'ACTIVE'` | `ACTIVE` / `INACTIVE` / `MAINTENANCE`              |
| `created_at`     | `TIMESTAMP`     | `DEFAULT NOW()`    |                                                    |
| `updated_at`     | `TIMESTAMP`     | `DEFAULT NOW()`    |                                                    |

**`court_images`**

| Cột          | Kiểu dữ liệu   | Ràng buộc                   | Mô tả            |
| ------------ | -------------- | --------------------------- | ---------------- |
| `id`         | `UUID`         | `PRIMARY KEY`               |                  |
| `court_id`   | `UUID`         | `NOT NULL, FK → courts(id)` | Sân chứa ảnh này |
| `image_url`  | `VARCHAR(500)` | `NOT NULL`                  | URL ảnh trên S3  |
| `is_primary` | `BOOLEAN`      | `DEFAULT false`             | Ảnh bìa của sân  |
| `created_at` | `TIMESTAMP`    | `DEFAULT NOW()`             |                  |

**`bookings`**

| Cột            | Kiểu dữ liệu    | Ràng buộc                   | Mô tả                                               |
| -------------- | --------------- | --------------------------- | --------------------------------------------------- |
| `id`           | `UUID`          | `PRIMARY KEY`               | Mã định danh đặt sân                                |
| `user_id`      | `UUID`          | `NOT NULL, FK → users(id)`  | Người đặt sân                                       |
| `court_id`     | `UUID`          | `NOT NULL, FK → courts(id)` | Sân được đặt                                        |
| `start_time`   | `TIMESTAMP`     | `NOT NULL`                  | Thời điểm bắt đầu                                   |
| `end_time`     | `TIMESTAMP`     | `NOT NULL`                  | Thời điểm kết thúc                                  |
| `total_amount` | `DECIMAL(12,2)` | `NOT NULL`                  | Giá tại thời điểm đặt                               |
| `status`       | `VARCHAR(20)`   | `DEFAULT 'PENDING'`         | `PENDING` / `CONFIRMED` / `CANCELLED` / `COMPLETED` |
| `note`         | `TEXT`          |                             | Ghi chú của người đặt                               |
| `created_at`   | `TIMESTAMP`     | `DEFAULT NOW()`             |                                                     |
| `updated_at`   | `TIMESTAMP`     | `DEFAULT NOW()`             |                                                     |

> Ngăn đặt trùng sân: sử dụng exclusion constraint của PostgreSQL với `btree_gist` trên `(court_id, tsrange(start_time, end_time))` để từ chối các khung giờ chồng lấp ở tầng cơ sở dữ liệu.

**`payments`** — xem phần Kết quả của Hieu ở trên.

#### Sơ đồ ERD

{{<mermaid>}}
erDiagram
users {
UUID id PK
VARCHAR email UK
VARCHAR password_hash
VARCHAR full_name
VARCHAR phone
VARCHAR cognito_sub UK
VARCHAR role
VARCHAR avatar_url
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

---

### Luồng thanh toán

Tính năng thanh toán trải dài trên cả hai luồng triển khai. Bảng dưới đây ánh xạ từng bước với actor, thao tác DB và dịch vụ AWS tương ứng.

| Bước | Actor                     | Hành động                                        | Thao tác DB                                                                            | Dịch vụ AWS               |
| ---- | ------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------- | ------------------------- |
| 1    | Người dùng                | Tìm kiếm và chọn sân + khung giờ                 | `SELECT` từ `courts`, kiểm tra không trùng lịch trong `bookings`                       | EC2 (FastAPI)             |
| 2    | Người dùng                | Xác nhận đặt sân                                 | `INSERT` vào `bookings` với `status = 'PENDING'`                                       | EC2 (FastAPI)             |
| 3    | Người dùng                | Khởi tạo thanh toán                              | `INSERT` vào `payments` với `status = 'PENDING'`, trả về `checkout_url`                | EC2 (FastAPI)             |
| 4    | Người dùng                | Chuyển đến cổng thanh toán và hoàn tất giao dịch | —                                                                                      | Cổng thanh toán bên ngoài |
| 5    | Cổng thanh toán           | Gửi webhook callback                             | —                                                                                      | Amazon API Gateway        |
| 6    | Lambda (Xử lý thanh toán) | Xác thực webhook, cập nhật bản ghi thanh toán    | `UPDATE payments SET status = 'SUCCESS', transaction_id = ..., gateway_response = ...` | AWS Lambda                |
| 7    | Lambda (Xác nhận đặt sân) | Xác nhận đặt sân                                 | `UPDATE bookings SET status = 'CONFIRMED'`                                             | AWS Lambda                |
| 8    | SNS                       | Thông báo cho người dùng                         | —                                                                                      | Amazon SNS                |
| 9    | Người dùng                | Xem xác nhận                                     | `SELECT` từ `bookings` JOIN `payments`                                                 | EC2 (FastAPI)             |

**Khi thanh toán thất bại (Bước 6 trả về `FAILED`):**

- `payments.status` → `FAILED`
- `bookings.status` → `CANCELLED`
- SNS thông báo lỗi đến người dùng

**Vòng đời trạng thái:**

```
bookings.status:   PENDING ──► CONFIRMED ──► COMPLETED
                       └──────► CANCELLED

payments.status:   PENDING ──► SUCCESS
                       └──────► FAILED ──► (thử lại sẽ tạo bản ghi payments mới)
```

**Nguyên tắc thiết kế then chốt:** Bước 1–3 do EC2 monolith xử lý (ghi vào `bookings` và `payments`). Bước 5–8 do luồng serverless xử lý hoàn toàn (Lambda đọc và ghi vào cùng các bảng RDS). Hai luồng không bao giờ ghi vào cùng một bản ghi cùng lúc — monolith tạo bản ghi, Lambda cập nhật chúng — do đó không xảy ra xung đột ghi.
