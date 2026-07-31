---
title: "Best Practice cho sơ đồ kiến trúc"
date: 2026-07-31
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# Best Practice cho sơ đồ kiến trúc

Một sơ đồ kiến trúc tốt không phải để trang trí — nó là cách một thiết kế được truyền đạt, review và bảo vệ. Trong dự án này, những bản vẽ đầu tiên của chúng tôi bị mentor bác trong buổi review ("VPC đâu?", "sao GitHub lại nằm trong AWS?"), và việc sửa chúng dạy tôi rằng một sơ đồ cũng tuân theo quy tắc y như code. Blog này tổng hợp các best practice đã biến bản nháp bị bác thành một sơ đồ sạch, sẵn sàng review.

**Tham khảo:** [How to draw professional AWS architecture diagrams (draw.io)](https://youtu.be/l8isyDe-GwY)

---

### 1. Dùng bộ icon AWS chính thức, mới nhất

Trộn icon AWS cũ và mới — hay dùng bộ mặc định thiếu sót của công cụ — trông thiếu chuyên nghiệp với người review. Hãy tải **AWS Architecture Icons** chính thức và dùng nhất quán một thế hệ icon xuyên suốt. Trong draw.io, giữ mọi icon service ở **kích thước đồng nhất** (ví dụ 60 px) để sơ đồ nhìn cân đối.

### 2. Tôn trọng thứ bậc resource

Các container phải lồng nhau đúng thứ tự, vẽ dưới dạng group frame có nhãn:

```
AWS Cloud  →  Region  →  VPC  →  Availability Zone  →  Subnet  →  resource
```

Một resource nằm sai frame là lỗi tính đúng đắn, không phải lỗi thẩm mỹ. Người review đọc cấu trúc lồng nhau để hiểu service *thực sự* nằm ở đâu.

### 3. Đặt mỗi service đúng scope

Đây là lỗi phổ biến nhất trong buổi review của chúng tôi. Service thuộc các scope khác nhau, và sơ đồ phải thể hiện điều đó:

- **Global** (ngoài Region): IAM, Route 53, CloudFront.
- **Regional** (trong Region, ngoài VPC): Amplify, Cognito, S3, API Gateway, Lambda, SNS, SSM.
- **Gắn với VPC / subnet**: ALB và NAT (public subnet), EC2 và RDS (private subnet).

Hai quy tắc suy ra từ đây: vẽ **Availability Zone hơi lấn ra ngoài VPC** (chúng là hạ tầng vật lý, không phải thành phần con của VPC), và biểu diễn **Auto Scaling Group là một frame** bao quanh các instance — không bao giờ là một icon đơn.

### 4. Để các actor không thuộc AWS ra ngoài ranh giới AWS Cloud

Người dùng, developer, cổng thanh toán bên ngoài, và **GitHub** không phải resource AWS — chúng phải nằm *ngoài* frame AWS Cloud. Đặt GitHub bên trong AWS là một lỗi thật mà mentor đã chỉ ra trong bản nháp của chúng tôi.

### 5. Thể hiện đường đi của mạng

Người chấm phải lần theo được một request từ đầu đến cuối mà không phải đoán: user → Internet Gateway → ALB → EC2 → RDS. Thể hiện Internet Gateway trên ranh giới VPC, đường egress qua NAT Gateway cho private subnet, và replication của RDS giữa các AZ. Đánh số các bước và liệt kê trong một text box bên cạnh.

### 6. Ngôn ngữ hình ảnh nhất quán

- **Icon đồng kích thước**, và một legend giải thích mọi mũi tên và màu sắc.
- **Color-code viền** — ví dụ một màu cho service chính, một màu cho sub-feature — và cho mũi tên một ý nghĩa nhất quán (nét liền = luồng request, nét đứt = luồng vận hành/CI-CD).
- **Bảo vệ khả năng đọc**: cho text box nền trắng đặc để mũi tên/đường không xuyên qua nhãn.
- Dùng **"To Back" (Cmd/Ctrl+Shift+B)** để đẩy các group frame lớn ra sau icon, giúp icon vẫn click chọn được.

### 7. Tái sử dụng bằng custom library và repo nhóm

Lưu các thành phần đã format sẵn (hoặc cả một layout VPC) thành **custom library của draw.io** để không phải format lại mỗi lần. Chia sẻ library và file `.drawio` gốc trong một repo nhóm tập trung để mọi người theo cùng một style — đừng copy sơ đồ vu vơ trên internet.

---

### Điều rút ra

Bài học đọng lại với tôi: **sơ đồ kiến trúc là một artifact kỹ thuật có quy tắc đúng-sai, không phải một bức tranh.** Đặt đúng scope, thứ bậc và đường đi của mạng là điều giúp nó vượt qua review — và buộc bạn thực sự hiểu mỗi service trong thiết kế của mình nằm ở đâu.
