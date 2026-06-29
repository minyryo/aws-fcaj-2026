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

![Kiến trúc Hybrid Đặt sân - Tổng quan](/images/2-Proposal/court_booking_hybrid_v2.png)

![Kiến trúc Hybrid Đặt sân - Luồng chi tiết](/images/2-Proposal/court_booking_hybrid_v1.png)

### Dịch vụ AWS sử dụng

- **Amazon EC2 + Auto Scaling**: Chạy backend FastAPI; mở rộng ngang theo tải.
- **Elastic Load Balancer**: Phân phối lưu lượng người dùng đến các EC2 instance.
- **Amazon Cognito**: Quản lý danh tính người dùng và xác thực dựa trên token.
- **Amazon RDS (PostgreSQL)**: Nguồn dữ liệu duy nhất cho đặt sân, người dùng và sân.
- **Amazon S3**: Lưu trữ ảnh sân và tài nguyên tĩnh.
- **Amazon API Gateway**: Tiếp nhận yêu cầu thanh toán và chuyển đến Lambda.
- **AWS Lambda**: Xử lý thanh toán và ghi xác nhận đặt sân (hai hàm).
- **Amazon SNS**: Gửi thông báo xác nhận đặt sân đến người dùng.
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

### 6. Lộ trình & Mốc triển khai

| Tuần | Công việc                                                     |
| ---- | ------------------------------------------------------------- |
| 1–2  | Thiết lập VPC, EC2, RDS, Cognito; triển khai skeleton FastAPI |
| 3    | Cấu hình Auto Scaling và ELB; tích hợp xác thực Cognito       |
| 4    | Xây dựng logic đặt sân; đảm bảo ràng buộc giao dịch trên RDS  |
| 5    | Triển khai luồng thanh toán API Gateway + Lambda              |
| 6    | Kết nối thông báo SNS; Lambda xác nhận ghi vào RDS            |
| 7    | Thiết lập CloudWatch dashboard và cảnh báo; tích hợp S3       |
| 8    | Kiểm thử đầu cuối, tinh chỉnh hiệu suất, hoàn thiện tài liệu  |

### 7. Ước tính ngân sách

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

### 8. Đánh giá rủi ro

#### Ma trận rủi ro

- **Race condition đặt trùng sân**: Ảnh hưởng cao, xác suất thấp - giảm thiểu bằng giao dịch RDS và row-level locking.
- **EC2 instance lỗi**: Ảnh hưởng cao, xác suất thấp - giảm thiểu bằng Auto Scaling và health check của ELB.
- **Lambda cold start khi thanh toán**: Ảnh hưởng trung bình, xác suất thấp - giảm thiểu bằng provisioned concurrency nếu cần.
- **Vượt ngân sách**: Ảnh hưởng trung bình, xác suất thấp - giảm thiểu bằng cảnh báo billing CloudWatch và chính sách cooldown Auto Scaling.

#### Chiến lược giảm thiểu

- Sử dụng `SELECT FOR UPDATE` hoặc advisory lock của PostgreSQL để ngăn xung đột đặt sân đồng thời ở tầng cơ sở dữ liệu.
- Cấu hình health check ELB để tự động thay thế EC2 instance không hoạt động.
- Đặt cảnh báo AWS Budget để thông báo khi chi tiêu hàng tháng vượt ngưỡng định sẵn.

### 9. Kết quả kỳ vọng

- **Kỹ thuật**: Hệ thống đặt sân hoàn chỉnh với thông tin thời gian thực, thanh toán bảo mật và thông báo tức thì - triển khai trên kiến trúc AWS hybrid cấp độ production.
- **Học thuật**: Thể hiện kinh nghiệm thực hành với EC2, RDS, Cognito, Lambda, API Gateway, SNS, S3 và CloudWatch trong một dự án duy nhất.
- **Vận hành**: Auto Scaling đảm bảo hệ thống xử lý tốt các đợt tăng lưu lượng mà không cần can thiệp thủ công; CloudWatch cung cấp khả năng quan sát liên tục cho sức khỏe hệ thống.
