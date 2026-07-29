---
title: "Xác thực (Cognito)"
date: 2026-07-30
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

Amazon Cognito là **nguồn xác thực** của dự án: lưu user, kiểm tra password, phát hành JWT. Một quyết định thiết kế then chốt định hình mọi thứ ở đây:

{{% notice note %}}
**Frontend không bao giờ nói chuyện với Cognito.** SPA không dùng Amplify Auth SDK. Backend *bao bọc* Cognito (`app/cognito.py`): FE gọi `POST /api/v1/auth/login` trên API của chúng ta; BE gọi Cognito, trả token về; FE chỉ lưu token và gửi `Authorization: Bearer <token>` trong mọi request. Kết quả: frontend không cần bất kỳ cấu hình Cognito nào, và luồng auth có thể thay thế hoặc mock hoàn toàn phía server.
{{% /notice %}}

#### 1. User pool, app client, và group

Ba resource, tạo một lần:

```bash
# 1. Pool — email là username, tự xác minh
POOL_ID=$(aws cognito-idp create-user-pool --pool-name court-booking \
  --username-attributes email --auto-verified-attributes email \
  --query 'UserPool.Id' --output text --profile thanh)

# 2. App client PUBLIC cho SPA — không có secret (trình duyệt không giữ secret được)
CLIENT_ID=$(aws cognito-idp create-user-pool-client --user-pool-id $POOL_ID \
  --client-name court-booking-spa --no-generate-secret \
  --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH ALLOW_USER_SRP_AUTH \
  --query 'UserPoolClient.ClientId' --output text --profile thanh)

# 3. Group = role của ứng dụng
for g in player court_manager admin; do
  aws cognito-idp create-group --user-pool-id $POOL_ID --group-name "$g" --profile thanh
done
```

![Cognito setup](/images/5-Workshop/5.5/cognito_setup.png)

Ba group phản chiếu đúng mô hình role: group của user xuất hiện trong JWT dưới claim `cognito:groups`; backend cache nó vào `users.role`, nơi các dependency `require_role("admin")` / `require_manager` kiểm tra ở mọi endpoint được bảo vệ. Đổi role của một user trong production nghĩa là đổi **Cognito group trước**, rồi mới tới cache trong DB — Cognito luôn là source of truth.

#### 2. Nối với backend

Pool ID và client ID được đưa vào SSM (mục 5.4) và đến app qua cây cầu env:

```bash
aws ssm put-parameter --name /court-booking/dev/COGNITO_USER_POOL_ID --type String --value "$POOL_ID"   --overwrite --profile thanh
aws ssm put-parameter --name /court-booking/dev/COGNITO_CLIENT_ID    --type String --value "$CLIENT_ID" --overwrite --profile thanh
```

Cơ chế chuyển đổi là tự động: khi `APP_ENV=local`, `auth.py` dùng **fallback chế độ local** (Bearer token được so trực tiếp với các giá trị `cognito_sub` trong seed — chính điều này cho phép walking-skeleton deploy ở mục 5.6 chạy trước khi Cognito tồn tại). Khi `APP_ENV=dev` và hai ID có mặt, `cognito.py` gọi pool thật — cùng endpoint, cùng code frontend, JWT thật.

{{% notice tip %}}
Chủ động để lại sau: Google OAuth (`POST /auth/oauth`) cần thêm **hosted-UI domain + identity provider** của Cognito. Auth email/password không cần gì trong số đó, nên social login là một bước độc lập, làm sau.
{{% /notice %}}
