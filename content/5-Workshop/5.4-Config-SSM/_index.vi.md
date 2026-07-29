---
title: "Cấu hình & Secret (SSM)"
date: 2026-07-30
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

Không có cấu hình nào nằm trong repo, trong GitHub, hay nướng sẵn vào AMI. **SSM Parameter Store là nguồn duy nhất cho config runtime**, dưới một path cho mỗi môi trường:

```
/court-booking/dev/APP_ENV                 String        dev
/court-booking/dev/DATABASE_URL            SecureString  postgresql+psycopg://app:…@<rds-endpoint>:5432/courtbooking
/court-booking/dev/COGNITO_USER_POOL_ID    String        (từ mục 5.5)
/court-booking/dev/COGNITO_CLIENT_ID       String        (từ mục 5.5)
/court-booking/dev/CORS_ORIGINS            String        ["https://<amplify-url>"]
```

Secret (DB URL, có nhúng password) dùng **SecureString** — mã hóa at rest bằng KMS; các định danh thường dùng `String`.

![Add SSM parameter](/images/5-Workshop/5.4/ssm_add_param_0.png)

![Add SSM parameter — SecureString](/images/5-Workshop/5.4/ssm_add_param_1.png)

#### Cây cầu: làm sao parameter đến được ứng dụng

App FastAPI đọc thiết lập từ **biến môi trường** (pydantic `BaseSettings` trong `config.py`) — nó không tự gọi SSM. Mối nối được thực hiện lúc **deploy**: hook `install` của CodeDeploy kéo toàn bộ parameter dưới path và ghi ra một env file để systemd nạp cho process:

```bash
# deploy/scripts/install.sh (chạy trên EC2 instance ở mỗi lần deploy)
umask 077
aws ssm get-parameters-by-path --path /court-booking/dev/ --with-decryption \
  --region "$REGION" --query 'Parameters[].[Name,Value]' --output text \
  | while read -r name value; do echo "${name##*/}=${value}"; done > /etc/court-booking.env
chmod 600 /etc/court-booking.env
```

```ini
# systemd unit
[Service]
EnvironmentFile=/etc/court-booking.env
ExecStart=/opt/court-booking/.venv/bin/gunicorn app.main:app -k uvicorn.workers.UvicornWorker -b 0.0.0.0:8000
```

**Tên lá** của parameter (`DATABASE_URL`) trở thành tên biến môi trường, và pydantic khớp nó với field `database_url` trong settings. Đổi cấu hình = cập nhật parameter + redeploy (hoặc restart); không sửa code, không build lại image.

{{% notice note %}}
Hai chi tiết quan trọng: **instance role** của EC2 cần `ssm:GetParameter*` trên path đó cộng `kms:Decrypt` (cho SecureString); và `CORS_ORIGINS` là field *list* trong pydantic, nên giá trị phải là một **chuỗi JSON array** (`["https://…"]`) — một URL trần sẽ parse lỗi.
{{% /notice %}}

{{% notice tip %}}
Tại sao không dùng GitHub Secrets? Chúng chỉ tồn tại bên trong workflow run. Parameter trong SSM thì *instance* đọc được lúc boot/deploy — nhờ vậy an toàn với Auto Scaling: một instance mới do ASG khởi chạy sẽ tự cấu hình mà không cần bất kỳ thao tác thủ công nào.
{{% /notice %}}
