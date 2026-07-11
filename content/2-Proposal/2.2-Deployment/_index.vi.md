---
title: "Quy trình triển khai (CI/CD)"
date: 2026-07-11
weight: 2
chapter: false
pre: " <b> 2.2. </b> "
---

# CI/CD — Thiết kế & Hướng dẫn thiết lập từng bước

Thông tin chung:

- **Phụ trách:** Hiếu (Backend — Repo & Triển khai, phân công 05/07/2026)
- **Đối tượng:** toàn nhóm (FE: Danh, Hùng · BE: Thành, Nguyên, Hiếu)
- **Stack:** React + Vite (TS) trên Amplify · FastAPI (Python) trên EC2/ASG · Lambda + SAM · RDS PostgreSQL

### 1. Tổng quan thiết kế CI/CD

**Nguyên tắc thiết kế:** GitHub Actions là bộ điều phối duy nhất; các dịch vụ AWS-native là đích triển khai. Một monorepo, ba đường triển khai tương ứng ba vùng kiến trúc:

```
                        ┌─────────────── GitHub monorepo ───────────────┐
                        │  /frontend   /backend    /lambdas   /infra    │
                        └──────┬────────────┬──────────┬────────────────┘
        Mở PR:            lint + test   lint + test  lint + test   (lọc theo path)
                               │            │             │
        Merge vào main:        ▼            ▼             ▼
                        ┌────────────┐ ┌─────────────┐ ┌──────────────┐
                        │  Amplify   │ │ GH Actions  │ │ GH Actions   │
                        │  CI/CD     │ │ → CodeDeploy│ │ → SAM deploy │
                        │  tích hợp  │ │ → EC2 ASG   │ │ → Lambda ×2  │
                        └────────────┘ └─────────────┘ │ + API GW     │
                                                       └──────────────┘
```

| Thành phần                         | Pipeline                                                          | Lý do                                                                                                                                                               |
| ---------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Frontend** (React + Vite)        | CI/CD tích hợp sẵn của Amplify Hosting                            | Đã có trong sơ đồ kiến trúc; build miễn phí, tự động tạo **URL preview cho mỗi PR** phục vụ review UI, rollback đơn giản, không cần bảo trì YAML.                   |
| **Backend** (FastAPI trên EC2)     | GitHub Actions → AWS CodeDeploy → rolling deploy trên ASG         | CodeDeploy **nhận biết Auto Scaling**: instance mà ASG khởi chạy sau đó tự động nhận bản mới nhất. Rollout có kiểm tra sức khỏe ELB = zero downtime với 2 instance. |
| **Lambda** (thanh toán + xác nhận) | GitHub Actions → AWS SAM                                          | Cả hai hàm + API Gateway + SNS trong một `template.yaml` — hạ tầng dưới dạng mã, có thể review.                                                                     |
| **Migration DB** (Alembic)         | Bước pipeline chạy trước khi deploy backend, chạy một lần qua SSM | Ngăn lệch schema; exclusion constraint nằm trong migration có phiên bản.                                                                                            |

**Các quyết định xuyên suốt:**

1. **GitHub OIDC → IAM role** — không lưu AWS key dài hạn trong GitHub secrets; mỗi pipeline một role tối thiểu quyền.
2. **Monorepo với path filter** — PR frontend không bao giờ kích hoạt deploy backend; một luồng review cho nhóm 5 người.
3. **Hai môi trường, có cổng phê duyệt** — merge tự động deploy lên dev; prod chờ phê duyệt qua GitHub Environment.
4. **Quality gate trên mỗi PR** — `ruff` + `pytest` (+ Postgres service container), `tsc` + `eslint` + `vitest`.

#### Các phương án đã cân nhắc — và lý do không chọn

**AWS CodePipeline + CodeBuild toàn phần** sẽ là câu trả lời "thuần AWS", nhưng: CodeBuild tính phí theo phút build trong khi GitHub Actions miễn phí cho repo public/giáo dục, trải nghiệm phát triển kém mượt hơn (dò log trong console thay vì check hiển thị ngay trong PR), và nhóm vốn đã làm việc trên GitHub. Ta vẫn khai thác sâu các dịch vụ AWS qua CodeDeploy + SAM + Amplify + OIDC/IAM — có thể nói là kết hợp tốt nhất của cả hai thế giới, và mỗi dịch vụ đó đều là kỹ năng AWS giá trị trong hồ sơ.

**Container trên ECS/Fargate thay vì EC2 + CodeDeploy.** Container hóa FastAPI sẽ hiện đại hóa việc triển khai (image bất biến, đồng nhất local/prod), nhưng nó thay thế chính EC2 monolith đã cam kết trong [Thiết kế kiến trúc](../2.1-architecture/) — tức là đổi kiến trúc compute, không chỉ đổi pipeline. Nó cũng thêm Docker, ECR và ECS thành ba thứ mới phải học trong khung 8 tuần, mà không mang lại lợi ích chức năng nào ở quy mô này. Đáng cân nhắc lại sau chương trình nếu ứng dụng tiếp tục phát triển.

**CDK hoặc Terraform thay vì SAM cho luồng serverless.** Terraform kéo theo quản lý state của bên thứ ba, đi ngược nguyên tắc "tối đa hóa dịch vụ AWS". CDK rất mạnh nhưng là framework IaC tổng quát — nặng hơn để học so với template khai báo của SAM, trong khi nhu cầu chỉ đúng hai hàm, một HTTP API và một topic SNS. SAM là công cụ nhỏ nhất làm được việc, và việc chuyển SAM → CDK sau này khá đơn giản.

**GitHub Actions → S3 + CloudFront cho frontend thay vì Amplify.** Kiểm soát nhiều hơn (chính sách cache, invalidation), nhưng mọi thứ Amplify cho miễn phí sẽ phải tự xây — quan trọng nhất là URL preview cho từng PR, vốn là toàn bộ vòng review của nhóm FE.

#### Lưu ý thẳng thắn — cần biết trước khi bắt đầu

1. **Việc cài đặt CodeDeploy agent trên EC2 có độ dốc học tập** (cài agent trong user-data của launch template, các hook `appspec.yml`). Dự trù nửa ngày cho việc này. Phương án dự phòng nếu gặp trục trặc: GitHub Actions → SSM `RunCommand` với script deploy — đơn giản hơn, nhưng mất khả năng nhận biết ASG và rollback tự nhiên, nên chỉ coi là phương án B.
2. **Lambda trong VPC cần đường ra internet.** Hàm xử lý thanh toán nằm trong subnet private để truy cập RDS — nhưng nếu nó phải _chủ động gọi ra_ API của cổng thanh toán (xác minh chữ ký, tra cứu trạng thái giao dịch), subnet private không có route internet. Các lựa chọn: NAT gateway (~$32/tháng — gần gấp đôi ngân sách dự án), NAT instance trên t3.micro (thân thiện free-tier, tốn công vận hành hơn), hoặc thiết kế luồng webhook sao cho Lambda không bao giờ khởi tạo cuộc gọi ra ngoài (xác thực payload đến bằng chữ ký HMAC). Hãy quyết định điều này **trước khi** cấu hình subnet/SG trong template SAM.
3. **Deploy CloudFormation không tức thời.** `sam deploy` chạy changeset — dự kiến 1–3 phút kể cả cho thay đổi một dòng ở Lambda. Điều đó bình thường; đừng cố tối ưu.
4. **Bước migration giả định có ít nhất một instance đang chạy khỏe mạnh.** Bước SSM nhắm vào instance đầu tiên đang chạy có tag phù hợp — ổn ở quy mô này, nhưng là giả định cần biết khi debug lần deploy đầu tiên vào một ASG trống.

> Các placeholder dùng xuyên suốt — thay toàn cục trước khi chạy bất kỳ lệnh nào:
> `<ACCOUNT_ID>` (mã tài khoản AWS 12 số), `<REGION>` (ví dụ `ap-southeast-1`), `<GH_ORG>/<GH_REPO>`.

---

### 2. Khởi tạo Repository

#### 2.1 Tạo monorepo

```bash
gh repo create <GH_ORG>/court-booking --private --clone
cd court-booking
mkdir -p frontend backend/app backend/alembic backend/deploy/scripts lambdas/process_payment lambdas/confirm_booking .github/workflows
```

Cấu trúc mục tiêu:

```
court-booking/
├── frontend/                    # React + Vite + TS (Danh, Hùng)
│   ├── src/
│   └── package.json
├── backend/                     # FastAPI monolith (Thành, Nguyên, Hiếu)
│   ├── app/                     #   mã ứng dụng
│   ├── alembic/                 #   migration DB
│   ├── deploy/
│   │   ├── appspec.yml          #   đặc tả CodeDeploy (copy vào gốc bundle khi deploy)
│   │   └── scripts/             #   các script lifecycle hook
│   ├── pyproject.toml
│   └── requirements.txt
├── lambdas/                     # Luồng thanh toán serverless (Hiếu)
│   ├── process_payment/app.py
│   ├── confirm_booking/app.py
│   ├── template.yaml            #   SAM: 2 hàm + API GW + SNS
│   └── samconfig.toml
├── amplify.yml                  # Build spec Amplify (frontend)
└── .github/workflows/
    ├── ci.yml                   # kiểm tra PR (lọc theo path)
    ├── deploy-backend.yml       # main → CodeDeploy → EC2 ASG
    └── deploy-lambdas.yml       # main → SAM deploy
```

#### 2.2 Bảo vệ nhánh

GitHub → repo → **Settings → Branches → Add rule** cho `main`:

- Yêu cầu pull request trước khi merge (1 phê duyệt)
- Yêu cầu status check pass — chọn `ci / frontend`, `ci / backend`, `ci / lambdas` (sau lần chạy CI đầu tiên)
- Không cho phép bỏ qua các thiết lập trên

#### 2.3 GitHub Environments (cổng phê duyệt deploy)

GitHub → **Settings → Environments**:

1. Tạo `dev` — không quy tắc bảo vệ (tự động deploy khi merge).
2. Tạo `prod` — Required reviewers → thêm Hiếu. Mọi deploy prod sẽ chờ một cú nhấp phê duyệt.

---

### 3. Chuẩn bị AWS (một lần)

#### 3.1 OIDC identity provider (không lưu AWS key trong GitHub!)

GitHub Actions sẽ assume IAM role qua OpenID Connect. Tạo provider một lần:

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

#### 3.2 Deploy role — pipeline backend

`trust-backend.json` — chỉ workflow từ _đúng repo này_ mới assume được:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<GH_ORG>/<GH_REPO>:*"
        }
      }
    }
  ]
}
```

```bash
aws iam create-role --role-name gh-deploy-backend \
  --assume-role-policy-document file://trust-backend.json
```

Policy quyền hạn (tối thiểu quyền — bucket S3 chứa revision + CodeDeploy + SSM cho migration):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::court-booking-deploy-<ACCOUNT_ID>/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codedeploy:CreateDeployment",
        "codedeploy:GetDeployment",
        "codedeploy:GetDeploymentConfig",
        "codedeploy:RegisterApplicationRevision",
        "codedeploy:GetApplicationRevision"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["ssm:SendCommand", "ssm:GetCommandInvocation"],
      "Resource": "*"
    }
  ]
}
```

```bash
aws iam put-role-policy --role-name gh-deploy-backend \
  --policy-name deploy-permissions --policy-document file://policy-backend.json
```

#### 3.3 Deploy role — pipeline lambda

Trust policy tương tự → role `gh-deploy-lambdas`. Gán quyền cho SAM/CloudFormation deploy (CloudFormation trên stack `court-booking-payment*`, `iam:PassRole` cho các role hàm do SAM tạo, S3 cho bucket artifact của SAM, cùng quyền ghi Lambda/APIGW/SNS). Với phạm vi dự án 8 tuần, `PowerUserAccess` giới hạn bằng permissions boundary là lối tắt chấp nhận được — siết chặt sau.

#### 3.4 Bucket artifact

```bash
aws s3 mb s3://court-booking-deploy-<ACCOUNT_ID> --region <REGION>
aws s3api put-bucket-versioning --bucket court-booking-deploy-<ACCOUNT_ID> \
  --versioning-configuration Status=Enabled
```

#### 3.5 Secret runtime trong SSM Parameter Store

Không bao giờ đặt thông tin DB trong GitHub. Ứng dụng đọc từ SSM khi khởi động:

```bash
aws ssm put-parameter --name /court-booking/dev/DATABASE_URL \
  --type SecureString --value "postgresql://app:<PASSWORD>@<RDS_ENDPOINT>:5432/courtbooking_dev"
aws ssm put-parameter --name /court-booking/prod/DATABASE_URL \
  --type SecureString --value "postgresql://app:<PASSWORD>@<RDS_ENDPOINT>:5432/courtbooking"
# tương tự: /court-booking/<env>/COGNITO_USER_POOL_ID, /COGNITO_CLIENT_ID, /SNS_TOPIC_ARN ...
```

---

### 4. Frontend — Amplify Hosting (CI/CD tích hợp)

#### 4.1 Build spec

`amplify.yml` ở gốc repo (hỗ trợ monorepo):

```yaml
version: 1
applications:
  - appRoot: frontend
    frontend:
      phases:
        preBuild:
          commands:
            - npm ci
        build:
          commands:
            - npm run build
      artifacts:
        baseDirectory: dist
        files:
          - "**/*"
      cache:
        paths:
          - node_modules/**/*
```

#### 4.2 Kết nối repo

1. Console → **Amplify → Create new app → GitHub** → cấp quyền → chọn `<GH_ORG>/court-booking`, nhánh `main`.
2. Amplify tự nhận `amplify.yml`; đặt **App root** = `frontend` (hộp thoại monorepo).
3. **Environment variables**: `VITE_API_URL = https://<ELB_DNS_hoặc_domain>/api/v1`.
4. **App settings → Previews → Enable pull request previews** trên `main` — mỗi PR có URL riêng (vòng review của Danh & Hùng).
5. Định tuyến SPA — **Rewrites and redirects**, thêm rule 200 (Rewrite) từ
   `</^[^.]+$|\.(?!(css|js|map|json|png|svg|jpg|ico|txt|woff2?)$)([^.]+$)/>` về `/index.html`.

**Kiểm chứng:** push một commit chạm `frontend/` → Amplify console hiển thị build → URL hosting phục vụ ứng dụng. Mở PR → URL preview xuất hiện trong PR checks.

---

### 5. Backend — EC2/ASG qua CodeDeploy

#### 5.1 Role cho EC2 instance

Instance cần _kéo_ revision và _đọc_ tham số SSM. Tạo role `court-booking-ec2` (trust EC2) với:

- `AmazonSSMManagedInstanceCore` (Session Manager + Run Command — cũng dùng cho migration)
- Inline: `s3:GetObject` trên `arn:aws:s3:::court-booking-deploy-<ACCOUNT_ID>/*`
- Inline: `ssm:GetParameter*` trên `arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/court-booking/*` + `kms:Decrypt` trên key SSM mặc định

Gán làm **instance profile** trong launch template.

#### 5.2 User-data của launch template (CodeDeploy agent + runtime ứng dụng)

```bash
#!/bin/bash
set -euo pipefail
dnf install -y ruby wget python3.12 python3.12-pip     # Amazon Linux 2023
cd /home/ec2-user
wget https://aws-codedeploy-<REGION>.s3.<REGION>.amazonaws.com/latest/install
chmod +x ./install && ./install auto
systemctl enable codedeploy-agent --now
useradd -r -s /sbin/nologin fastapi || true
mkdir -p /opt/court-booking && chown fastapi:fastapi /opt/court-booking
```

> Agent poll CodeDeploy ngay khi boot — đây là điều làm cho triển khai **an toàn với Auto Scaling**: mọi instance mà ASG khởi chạy sau này đều tự cài bản mới nhất trước khi vào phục vụ.

Unit systemd `/etc/systemd/system/court-booking.service` (được hook deploy cài đặt):

```ini
[Unit]
Description=Court Booking FastAPI
After=network.target

[Service]
User=fastapi
WorkingDirectory=/opt/court-booking
Environment=APP_ENV=prod
ExecStart=/opt/court-booking/.venv/bin/gunicorn app.main:app \
  -k uvicorn.workers.UvicornWorker -w 2 -b 0.0.0.0:8000
Restart=always

[Install]
WantedBy=multi-user.target
```

#### 5.3 CodeDeploy application + deployment group

```bash
aws deploy create-application --application-name court-booking-backend

# service role cho chính CodeDeploy
aws iam create-role --role-name codedeploy-service \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"codedeploy.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name codedeploy-service \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole

aws deploy create-deployment-group \
  --application-name court-booking-backend \
  --deployment-group-name prod \
  --service-role-arn arn:aws:iam::<ACCOUNT_ID>:role/codedeploy-service \
  --auto-scaling-groups court-booking-asg \
  --deployment-config-name CodeDeployDefault.OneAtATime \
  --load-balancer-info targetGroupInfoList=[{name=court-booking-tg}]
```

`OneAtATime` + tích hợp target group = rolling deploy: từng instance được gỡ khỏi ELB, cập nhật, kiểm tra sức khỏe, đăng ký lại — **zero downtime với 2 instance**.

#### 5.4 appspec.yml + các script hook

`backend/deploy/appspec.yml`:

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /opt/court-booking
    overwrite: true
file_exists_behavior: OVERWRITE
hooks:
  ApplicationStop:
    - location: deploy/scripts/stop.sh
      timeout: 60
  AfterInstall:
    - location: deploy/scripts/install.sh
      timeout: 300
  ApplicationStart:
    - location: deploy/scripts/start.sh
      timeout: 60
  ValidateService:
    - location: deploy/scripts/validate.sh
      timeout: 120
```

`backend/deploy/scripts/stop.sh`

```bash
#!/bin/bash
systemctl stop court-booking || true
```

`backend/deploy/scripts/install.sh`

```bash
#!/bin/bash
set -euo pipefail
cd /opt/court-booking
python3.12 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install -r requirements.txt
cp deploy/court-booking.service /etc/systemd/system/court-booking.service
systemctl daemon-reload
chown -R fastapi:fastapi /opt/court-booking
```

`backend/deploy/scripts/start.sh`

```bash
#!/bin/bash
systemctl start court-booking
```

`backend/deploy/scripts/validate.sh`

```bash
#!/bin/bash
set -euo pipefail
for i in $(seq 1 24); do
  curl -fsS http://localhost:8000/api/v1/health && exit 0
  sleep 5
done
echo "health check failed" >&2; exit 1
```

> `validate.sh` thất bại ⇒ CodeDeploy đánh dấu deployment thất bại và (khi bật rollback) tự quay về revision trước. Hãy triển khai `GET /api/v1/health` trong FastAPI ngay từ ngày đầu — đây cũng là đường health-check của ELB.

#### 5.5 Workflow GitHub Actions

`.github/workflows/deploy-backend.yml`:

```yaml
name: deploy-backend
on:
  push:
    branches: [main]
    paths: ["backend/**"]

permissions:
  id-token: write # OIDC
  contents: read

concurrency: deploy-backend # không bao giờ chạy hai deploy cùng lúc

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r backend/requirements.txt -r backend/requirements-dev.txt
      - run: ruff check backend
      - run: pytest backend

  deploy:
    needs: test
    runs-on: ubuntu-latest
    environment: prod # chờ phê duyệt (mục 2.3)
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/gh-deploy-backend
          aws-region: <REGION>

      - name: Bundle revision
        run: |
          cd backend
          cp deploy/appspec.yml .
          zip -r ../backend-${{ github.sha }}.zip . -x "**/__pycache__/*" ".venv/*"
      - name: Upload to S3
        run: aws s3 cp backend-${{ github.sha }}.zip s3://court-booking-deploy-<ACCOUNT_ID>/backend/

      - name: Run DB migrations (một lần, qua SSM trên một instance)
        run: |
          IID=$(aws ec2 describe-instances \
            --filters "Name=tag:app,Values=court-booking" "Name=instance-state-name,Values=running" \
            --query "Reservations[0].Instances[0].InstanceId" --output text)
          CMD_ID=$(aws ssm send-command --instance-ids "$IID" \
            --document-name AWS-RunShellScript \
            --parameters commands='cd /opt/court-booking && .venv/bin/alembic upgrade head' \
            --query Command.CommandId --output text)
          aws ssm wait command-executed --command-id "$CMD_ID" --instance-id "$IID"
          aws ssm get-command-invocation --command-id "$CMD_ID" --instance-id "$IID" \
            --query "{status:Status,out:StandardOutputContent,err:StandardErrorContent}"

      - name: Create CodeDeploy deployment
        run: |
          DID=$(aws deploy create-deployment \
            --application-name court-booking-backend \
            --deployment-group-name prod \
            --s3-location bucket=court-booking-deploy-<ACCOUNT_ID>,key=backend/backend-${{ github.sha }}.zip,bundleType=zip \
            --query deploymentId --output text)
          aws deploy wait deployment-successful --deployment-id "$DID"
```

> **Quy tắc thứ tự migration:** migration chạy _trước_ khi code mới được deploy, nên mọi migration phải **tương thích ngược với code đang chạy** (thêm cột dạng nullable, không bao giờ drop-rồi-rename trong cùng một release). Đây là kỷ luật duy nhất nhóm cần học về zero-downtime deploy.

**Kiểm chứng:** merge một thay đổi nhỏ dưới `backend/` → Actions chạy test → chờ tại cổng `prod` → phê duyệt → CodeDeploy console hiển thị rollout OneAtATime → `curl https://<ELB_DNS>/api/v1/health` trả về 200.

---

### 6. Lambda — Pipeline SAM

#### 6.1 Template SAM

`lambdas/template.yaml` (khung — hai hàm từ kiến trúc, gắn VPC để truy cập RDS):

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: Court booking - serverless payment path

Parameters:
  Env: { Type: String, AllowedValues: [dev, prod], Default: dev }

Globals:
  Function:
    Runtime: python3.12
    Timeout: 15
    MemorySize: 256
    VpcConfig:
      SecurityGroupIds: [!Ref LambdaSG] # SG được phép vào SG của RDS cổng 5432
      SubnetIds: [subnet-xxxx, subnet-yyyy] # subnet private
    Environment:
      Variables:
        DATABASE_URL_PARAM: !Sub /court-booking/${Env}/DATABASE_URL

Resources:
  PaymentApi:
    Type: AWS::Serverless::HttpApi

  ProcessPayment:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: process_payment/
      Handler: app.handler
      Events:
        Webhook:
          Type: HttpApi
          Properties:
            ApiId: !Ref PaymentApi
            Path: /api/v1/payments/webhook
            Method: POST
      Policies:
        - SSMParameterReadPolicy: { ParameterName: court-booking/* }
        - LambdaInvokePolicy: { FunctionName: !Ref ConfirmBooking }

  ConfirmBooking:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: confirm_booking/
      Handler: app.handler
      Policies:
        - SSMParameterReadPolicy: { ParameterName: court-booking/* }
        - SNSPublishMessagePolicy:
            { TopicName: !GetAtt BookingNotifications.TopicName }

  BookingNotifications:
    Type: AWS::SNS::Topic

  LambdaSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: court-booking lambdas
      VpcId: vpc-xxxx

Outputs:
  WebhookUrl:
    Value: !Sub "https://${PaymentApi}.execute-api.${AWS::Region}.amazonaws.com/api/v1/payments/webhook"
```

`lambdas/samconfig.toml`:

```toml
version = 0.1
[prod.deploy.parameters]
stack_name        = "court-booking-payment-prod"
resolve_s3        = true
region            = "<REGION>"
capabilities      = "CAPABILITY_IAM"
parameter_overrides = "Env=prod"
```

#### 6.2 Workflow

`.github/workflows/deploy-lambdas.yml`:

```yaml
name: deploy-lambdas
on:
  push:
    branches: [main]
    paths: ["lambdas/**"]

permissions: { id-token: write, contents: read }
concurrency: deploy-lambdas

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r lambdas/requirements-dev.txt
      - run: pytest lambdas

  deploy:
    needs: test
    runs-on: ubuntu-latest
    environment: prod
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/setup-sam@v2
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/gh-deploy-lambdas
          aws-region: <REGION>
      - run: sam build
        working-directory: lambdas
      - run: sam deploy --config-env prod --no-confirm-changeset --no-fail-on-empty-changeset
        working-directory: lambdas
```

**Kiểm chứng:** merge một thay đổi dưới `lambdas/` → stack `court-booking-payment-prod` cập nhật trong CloudFormation → `curl -X POST <WebhookUrl>` với payload thử → kiểm tra log group CloudWatch của hàm.

---

### 7. Kiểm tra PR (quality gate)

`.github/workflows/ci.yml` — chạy trên mỗi PR, lọc theo path nên chỉ thành phần bị ảnh hưởng mới build:

```yaml
name: ci
on:
  pull_request:

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.f.outputs.frontend }}
      backend: ${{ steps.f.outputs.backend }}
      lambdas: ${{ steps.f.outputs.lambdas }}
    steps:
      - uses: actions/checkout@v4
      - id: f
        uses: dorny/paths-filter@v3
        with:
          filters: |
            frontend: ["frontend/**"]
            backend:  ["backend/**"]
            lambdas:  ["lambdas/**"]

  frontend:
    needs: changes
    if: needs.changes.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          {
            node-version: 20,
            cache: npm,
            cache-dependency-path: frontend/package-lock.json,
          }
      - run: npm ci
        working-directory: frontend
      - run: npx tsc --noEmit && npx eslint src && npx vitest run
        working-directory: frontend

  backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: courtbooking_test }
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready" --health-interval 5s
          --health-timeout 5s --health-retries 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r backend/requirements.txt -r backend/requirements-dev.txt
      - run: ruff check backend
      - env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/courtbooking_test
        run: |
          cd backend
          alembic upgrade head      # migration cũng được test trên mỗi PR
          pytest

  lambdas:
    needs: changes
    if: needs.changes.outputs.lambdas == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r lambdas/requirements-dev.txt && pytest lambdas
```

> Postgres service container trên PR đồng nghĩa **exclusion constraint và logic row-locking được kiểm thử trên mỗi PR** — cơ chế chống đặt trùng sân được test liên tục.

---

### 8. Quy trình Rollback

| Thành phần        | Cách rollback                                                                                                                                                            | Thời gian      |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------- |
| Frontend          | Amplify console → app → **Redeploy** một bản build thành công trước đó                                                                                                   | ~2 phút        |
| Backend           | CodeDeploy console → deployments → revision trước → **Retry/redeploy**; hoặc `aws deploy create-deployment` trỏ về `backend-<sha>.zip` trước đó (bucket có versioning)   | ~5 phút        |
| Backend (tự động) | `validate.sh` thất bại kích hoạt rollback tự động khi bật: `aws deploy update-deployment-group ... --auto-rollback-configuration enabled=true,events=DEPLOYMENT_FAILURE` | tự động        |
| Lambda            | `git revert` commit merge → pipeline deploy lại template/code trước; hoặc CloudFormation console → stack → change set trước                                              | ~5 phút        |
| Migration DB      | `alembic downgrade -1` qua SSM Run Command — **chỉ khi migration có thể đảo ngược**; ưu tiên sửa tiến (roll-forward)                                                     | tùy trường hợp |

---

### 9. Thứ tự thiết lập & Phân công

| #   | Công việc                                                  | Phụ trách      | Phụ thuộc        |
| --- | ---------------------------------------------------------- | -------------- | ---------------- |
| 1   | Tạo monorepo, bảo vệ nhánh, environments (§2)              | Hiếu           | —                |
| 2   | OIDC provider + 2 deploy role + bucket artifact (§3)       | Hiếu           | Tài khoản AWS    |
| 3   | Tham số SSM (dev + prod)                                   | Hiếu           | Endpoint RDS     |
| 4   | Commit `ci.yml` — PR check hoạt động trước mọi mã ứng dụng | Hiếu           | 1                |
| 5   | Kết nối Amplify + PR preview (§4)                          | Danh & Hùng    | 1                |
| 6   | User-data launch template + instance role (§5.1–5.2)       | Hiếu           | VPC/ASG sẵn sàng |
| 7   | CodeDeploy app + deployment group (§5.3)                   | Hiếu           | 6                |
| 8   | `appspec.yml` + hooks + endpoint `/health` (§5.4)          | Thành & Nguyên | Khung FastAPI    |
| 9   | `deploy-backend.yml` chạy xanh lần đầu (§5.5)              | Hiếu           | 7, 8             |
| 10  | Template SAM + `deploy-lambdas.yml` (§6)                   | Hiếu           | 2                |
| 11  | Diễn tập rollback — thực hành từng dòng của §8 một lần     | cả nhóm        | 9, 10            |

---

### 10. Tác động chi phí

| Dịch vụ                        | Chi phí phát sinh                                                         |
| ------------------------------ | ------------------------------------------------------------------------- |
| GitHub Actions                 | $0 (free tier: 2.000 phút/tháng repo private; không giới hạn repo public) |
| Amplify Hosting build          | free tier 1.000 phút build/tháng, sau đó ~$0.01/phút                      |
| CodeDeploy (EC2)               | **$0** — miễn phí cho EC2/on-prem                                         |
| SAM / CloudFormation           | $0 (chỉ trả cho chính các Lambda)                                         |
| Bucket S3 artifact             | < $0.10/tháng                                                             |
| SSM Parameter Store (standard) | $0                                                                        |

Tổng: gần như **bằng không** so với ước tính vận hành ~$45/tháng trong [Thiết kế kiến trúc](../2.1-architecture/).
