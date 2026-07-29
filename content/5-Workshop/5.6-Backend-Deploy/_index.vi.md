---
title: "Deploy Backend (EC2 + CodeDeploy)"
date: 2026-07-30
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

Deploy backend là **một hành động duy nhất: `git push`**. Mọi thứ còn lại là tự động hóa. Mục này trình bày chuỗi deploy và các tài sản (asset) giúp nó vận hành.

#### 1. Chuỗi deploy

```
git push (backend/**)                                 developer
  → deploy-backend.yml                                GitHub Actions
  → OIDC token → AssumeRole gh-deploy-backend         không có AWS key (5.1)
  → zip revision → upload lên deploy S3 bucket
  → alembic upgrade head qua SSM Run Command          (migration DB, có điều kiện — xem §4)
  → aws deploy create-deployment                      CodeDeploy
      → agent trên EC2 kéo file zip từ S3
      → các lifecycle hook chạy (appspec.yml):
          ApplicationStop   deploy/scripts/stop.sh      systemctl stop
          AfterInstall      deploy/scripts/install.sh   venv + pip install,
                                                        SSM → /etc/court-booking.env (5.4),
                                                        cài systemd unit
          ApplicationStart  deploy/scripts/start.sh     systemctl start (gunicorn :8000)
          ValidateService   deploy/scripts/validate.sh  poll /api/v1/health đến khi 200
  → health check của ALB target group pass → instance nhận traffic
```

Instance không bao giờ giữ credential hay config trong AMI: **code đến từ S3, config đến từ SSM** — đều tại thời điểm deploy. Chính điều này làm setup an toàn với Auto Scaling: một instance mới do ASG khởi chạy sẽ chạy CodeDeploy agent, kéo revision mới nhất, và tự cấu hình.

#### 2. Điều kiện tiên quyết trên instance (user-data của launch template)

Các hook giả định bốn thứ đã tồn tại trên máy, được chuẩn bị một lần trong user-data: `python3.12`, system user `fastapi` (không login), **CodeDeploy agent**, và `/opt/court-booking`. Instance profile chỉ cấp quyền đọc SSM parameter (+ `kms:Decrypt`) và đọc S3 trên deploy bucket.

#### 3. Các asset deploy (commit trong `backend/deploy/`)

- `appspec.yml` — ánh xạ bundle vào `/opt/court-booking` và gắn bốn hook
- `scripts/stop.sh` — `systemctl stop court-booking || true` (chịu được lần deploy đầu tiên, khi service chưa tồn tại)
- `scripts/install.sh` — venv + `pip install -r requirements.txt`, cây cầu SSM→env, cài systemd unit
- `scripts/start.sh` / `scripts/validate.sh` — start, rồi poll `localhost:8000/api/v1/health` (24 × 5 s) trước khi CodeDeploy công nhận thành công
- `court-booking.service` — gunicorn với uvicorn worker, `User=fastapi`, `EnvironmentFile=/etc/court-booking.env`, `Restart=always`

{{% notice warning %}}
Hai lỗi được bắt **trước** khi chúng làm hỏng một lượt deploy: (1) hook script được commit mà thiếu execute bit — CodeDeploy chạy trực tiếp file và fail với `permission denied`; sửa bằng `git update-index --chmod=+x deploy/scripts/*.sh`. (2) `install.sh` copy một systemd unit chưa có trong repo — hook `AfterInstall` sẽ chết giữa chừng.
{{% /notice %}}

#### 4. Lần deploy đầu tiên: walking skeleton

Có một vấn đề con gà – quả trứng: workflow migrate DB **qua SSM trước khi** CodeDeploy đưa code xuống máy — nhưng ở lần deploy đầu tiên trên máy chưa có venv. Chúng tôi giải bằng trình tự:

1. **Deploy với `APP_ENV=local`** — app khởi động mà không chạm RDS; `/api/v1/health` trả 200. Một lượt deploy này chứng minh toàn bộ chuỗi (OIDC → S3 → CodeDeploy → hooks → systemd → ALB) khi rủi ro còn bằng không.
2. **Bootstrap DB từ bên trong VPC** — giờ máy đã có code + venv: chạy `alembic upgrade head` (schema + `btree_gist`) và nạp seed qua SSM Run Command. DB không public, nên đây là con đường *duy nhất* tới nó.
3. **Chuyển `APP_ENV` sang `dev`** trong SSM và redeploy — app chạy với RDS + Cognito thật. Từ đây mỗi lần push đều migrate tự động.

{{% notice tip %}}
Nguyên tắc **walking skeleton** ([v2 guide] §7): deploy thứ mỏng nhất có thể chạy trọn vẹn từ đầu đến cuối trước, rồi mới đắp thêm nội dung lên một pipeline đã hoạt động. Debug IAM, CodeDeploy và systemd cùng lúc với kết nối database khó hơn hẳn debug từng thứ một.
{{% /notice %}}
