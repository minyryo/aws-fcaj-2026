---
title: "Dọn dẹp tài nguyên"
date: 2026-07-30
weight: 8
chapter: false
pre: " <b> 5.8. </b> "
---

Khi dự án kết thúc, tài nguyên được gỡ theo **thứ tự phụ thuộc ngược** — ngược lại với lúc dựng. Chỉ những phần global, luôn miễn phí được giữ lại.

#### Thứ tự teardown

| # | Resource | Cách | Vì sao theo thứ tự này |
| - | --- | --- | --- |
| 1 | CodeDeploy app + deployment group | console / CLI | dừng các deployment đang nhắm vào instance |
| 2 | Auto Scaling Group (desired = 0) → launch template | EC2 console | terminate instance sạch sẽ trước khi SG của chúng bị xóa |
| 3 | ALB + target group | EC2 console | phí theo giờ dừng ngay khi xóa |
| 4 | RDS instance | `delete-db-instance --final-db-snapshot-identifier court-booking-final` | **chụp snapshot cuối** — giữ dữ liệu với chi phí ~$0 |
| 5 | Cognito user pool | console | dữ liệu user nằm trong RDS; pool tạo lại được |
| 6 | Amplify app | console | dừng build/hosting FE |
| 7 | S3 bucket (artifact deploy, assets) | empty rồi delete | bucket phải rỗng trước |
| 8 | SSM parameter | `delete-parameters` dưới `/court-booking/dev/` | xóa DB URL đã lưu (chứa password) |
| 9 | Security group + DB subnet group | VPC console | chỉ xóa được sau khi instance/RDS đã biến mất (nếu không sẽ gặp lỗi dependency) |

#### Những gì giữ lại (có chủ đích)

- **IAM role, OIDC provider, policy** — global, miễn phí, và là "hồ sơ" của thiết kế danh tính
- **Route 53 hosted zone** (account còn lại) — $0.50/tháng; giữ chừng nào còn sở hữu domain
- **Snapshot RDS cuối cùng** — bản sao dữ liệu duy nhất; xóa sau cùng, một cách có ý thức, hoặc share sang account kia trước (copy snapshot liên account) làm backup ngoài account

{{% notice warning %}}
Hai nguồn **phí chạy liên tục** trong stack này là **ALB** (theo giờ, ~$16–18/tháng ngoài free tier) và **RDS** (750 giờ miễn phí/tháng chỉ đủ một instance — instance thứ hai sẽ tính phí). Nếu chỉ tạm dừng dự án thay vì kết thúc: xóa ALB, stop RDS instance (tự khởi động lại sau 7 ngày), và đặt ASG về 0 — hóa đơn giảm xuống vài cent phí lưu trữ mà mọi thứ vẫn tạo lại được.
{{% /notice %}}

{{% notice tip %}}
Kiểm chứng account đã thực sự "im lặng": **Billing → Bills** phải cho thấy dự báo $0, và **Budgets alert** (tạo lúc setup) đóng vai trò dây bẫy nếu còn sót thứ gì.
{{% /notice %}}
