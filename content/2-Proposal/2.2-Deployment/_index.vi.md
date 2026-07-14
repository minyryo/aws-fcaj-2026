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

**Nguyên tắc thiết kế:** GitHub Actions là bộ điều phối duy nhất; các dịch vụ AWS-native là đích triển khai. Hai repository tách theo nhóm phụ trách — `court-booking-frontend` (Danh & Hùng) và `court-booking-backend` (FastAPI + Lambda, nhóm BE) — với ba đường triển khai tương ứng ba vùng kiến trúc:

```
              ┌─ court-booking-frontend ─┐   ┌────── court-booking-backend ──────┐
              │    React + Vite (TS)     │   │   /backend           /lambdas     │
              └────────────┬─────────────┘   └───────┬──────────────────┬────────┘
Mở PR:          lint + test + contract        lint + test          lint + test
                       check                     (lọc theo path trong repo)
Merge vào main:            ▼                          ▼                  ▼
              ┌────────────────┐             ┌─────────────┐    ┌──────────────┐
              │    Amplify     │             │ GH Actions  │    │ GH Actions   │
              │ CI/CD tích hợp │             │ → CodeDeploy│    │ → SAM deploy │
              └────────────────┘             │ → EC2 ASG   │    │ → Lambda ×2  │
                                             └─────────────┘    │ + API GW     │
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
2. **Hai repo tách theo nhóm phụ trách** — FE và BE release theo tiến độ riêng; nếu API contract giữa hai bên bị lệch (drift), job kiểm tra dựa trên OpenAPI trong repo frontend sẽ phát hiện ngay khi mở PR (§4.3). Bên trong repo backend, path filter giữ pipeline backend và lambda độc lập với nhau.
3. **Hai môi trường, có cổng phê duyệt** — merge tự động deploy lên dev; prod chờ phê duyệt qua GitHub Environment.
4. **Quality gate trên mỗi PR** — `ruff` + `pytest` (+ Postgres service container), `tsc` + `eslint` + `vitest`.

#### Các phương án đã cân nhắc — và lý do không chọn

**AWS CodePipeline + CodeBuild toàn phần** sẽ là câu trả lời "thuần AWS", nhưng: CodeBuild tính phí theo phút build trong khi GitHub Actions miễn phí cho repo public/giáo dục, trải nghiệm phát triển kém mượt hơn (dò log trong console thay vì check hiển thị ngay trong PR), và nhóm vốn đã làm việc trên GitHub. Nhóm vẫn được thực hành chuyên sâu các dịch vụ AWS qua CodeDeploy + SAM + Amplify + OIDC/IAM — tận dụng được ưu điểm của cả hai phía, và mỗi dịch vụ đó đều là kỹ năng AWS đáng giá trong CV.

**Một monorepo thay vì hai repo.** Một repo cho phép gói thay đổi của cả FE lẫn BE trong cùng một PR và chỉ có một luồng review — nghe rất hợp lý trên lý thuyết. Nhưng hai bạn FE và ba bạn BE gần như không đụng vào code của nhau, toolchain tách biệt hoàn toàn (npm và Python), và Amplify kết nối đơn giản hơn với một repo frontend riêng. Lợi thế duy nhất của monorepo thực sự quan trọng — giữ API contract đồng bộ giữa hai phía — đã được bù đắp bằng job sinh kiểu dữ liệu tự động từ OpenAPI (§4.3), nên tách repo theo nhóm phụ trách là lựa chọn hợp lý hơn.

**Container trên ECS/Fargate thay vì EC2 + CodeDeploy.** Container hóa FastAPI sẽ hiện đại hóa việc triển khai (image bất biến, đồng nhất local/prod), nhưng nó thay thế chính EC2 monolith đã cam kết trong [Thiết kế kiến trúc](../2.1-architecture/) — tức là đổi kiến trúc compute, không chỉ đổi pipeline. Nó cũng thêm Docker, ECR và ECS thành ba thứ mới phải học trong khung 8 tuần, mà không mang lại lợi ích chức năng nào ở quy mô này. Đáng cân nhắc lại sau chương trình nếu ứng dụng tiếp tục phát triển.

**Docker trên EC2 — giữ nguyên kiến trúc, chỉ đóng gói bằng container.** Khác với phương án ECS ở trên: thiết kế ASG + ELB + CodeDeploy giữ nguyên; chỉ có cách đóng gói thay đổi (CI build image, push lên Amazon ECR, và các hook CodeDeploy đổi `pip install`/`systemctl` thành `docker pull`/`docker run`). Lợi ích là có thật — một image duy nhất chấm dứt tình trạng lệch môi trường giữa 3 máy dev + CI + EC2, và deploy không còn chạy `pip install` trên instance đang phục vụ. Chưa áp dụng ngay vì nó thêm Docker, ECR và bẫy build khác kiến trúc trên máy Apple Silicon (`docker buildx --platform linux/amd64`) vào khối lượng phải học vốn đã đầy. **Quy tắc quyết định:** nếu nhóm mất hơn nửa ngày vì lệch môi trường ("chạy được trên máy mình, fail trên CI"), hãy chuyển sang Docker tại thời điểm đó — việc chuyển đổi rẻ chính vì CodeDeploy giữ nguyên. Trước đó, hai biện pháp chi phí thấp đã mang lại phần lớn lợi ích đồng nhất môi trường: commit file `.python-version` (§2.4, bước 2) và chạy Postgres local bằng Docker Compose (§2.4, bước 6).

**CDK hoặc Terraform thay vì SAM cho luồng serverless.** Terraform kéo theo quản lý state của bên thứ ba, đi ngược nguyên tắc "tối đa hóa dịch vụ AWS". CDK rất mạnh nhưng là framework IaC tổng quát — nặng hơn để học so với template khai báo của SAM, trong khi nhu cầu chỉ đúng hai hàm, một HTTP API và một topic SNS. SAM là công cụ nhỏ nhất làm được việc, và việc chuyển SAM → CDK sau này khá đơn giản.

**GitHub Actions → S3 + CloudFront cho frontend thay vì Amplify.** Kiểm soát nhiều hơn (chính sách cache, invalidation), nhưng mọi thứ Amplify cho miễn phí sẽ phải tự xây — quan trọng nhất là URL preview cho từng PR, vốn là toàn bộ vòng review của nhóm FE.

#### Một số lưu ý thực tế — cần biết trước khi bắt đầu

1. **Việc cài đặt CodeDeploy agent trên EC2 cần thời gian làm quen** (cài agent trong user-data của launch template, các hook `appspec.yml`). Dự trù nửa ngày cho việc này. Phương án dự phòng nếu gặp trục trặc: GitHub Actions → SSM `RunCommand` với script deploy — đơn giản hơn, nhưng mất khả năng nhận biết ASG và cơ chế rollback tích hợp sẵn, nên chỉ coi là phương án B.
2. **Lambda trong VPC cần đường ra internet.** Hàm xử lý thanh toán nằm trong subnet private để truy cập RDS — nhưng nếu nó phải _chủ động gọi ra_ API của cổng thanh toán (xác minh chữ ký, tra cứu trạng thái giao dịch), subnet private không có route internet. Các lựa chọn: NAT gateway (~$32/tháng — gần gấp đôi ngân sách dự án), NAT instance trên t3.micro (thân thiện free-tier, tốn công vận hành hơn), hoặc thiết kế luồng webhook sao cho Lambda không bao giờ khởi tạo cuộc gọi ra ngoài (xác thực payload đến bằng chữ ký HMAC). Hãy quyết định điều này **trước khi** cấu hình subnet/SG trong template SAM.
3. **Deploy CloudFormation không tức thời.** `sam deploy` chạy changeset — dự kiến 1–3 phút kể cả cho thay đổi một dòng ở Lambda. Điều đó bình thường; đừng cố tối ưu.
4. **Bước migration giả định có ít nhất một instance đang chạy khỏe mạnh.** Bước SSM nhắm vào instance đầu tiên đang chạy có tag phù hợp — ổn ở quy mô này, nhưng là giả định cần biết khi debug lần deploy đầu tiên vào một ASG trống.

> Các placeholder dùng xuyên suốt — thay toàn cục trước khi chạy bất kỳ lệnh nào:
> `<ACCOUNT_ID>` (mã tài khoản AWS 12 số), `<REGION>` (ví dụ `ap-southeast-1`), `<GH_ORG>` (tổ chức hoặc tên người dùng GitHub). Hai repo được đặt tên `court-booking-frontend` và `court-booking-backend`.

---

### 2. Khởi tạo Repository

#### 2.1 Tạo hai repository

Việc tách repo dựa theo phân công của nhóm: hai bạn FE phụ trách một repo, ba bạn BE phụ trách repo còn lại. Các Lambda nằm **cùng repo backend** — chúng dùng chung schema cơ sở dữ liệu, lịch sử migration Alembic và do cùng nhóm phát triển.

```bash
gh repo create <GH_ORG>/court-booking-frontend --private --clone
gh repo create <GH_ORG>/court-booking-backend  --private --clone

cd court-booking-frontend
mkdir -p src .github/workflows

cd ../court-booking-backend
mkdir -p backend/app backend/alembic backend/deploy/scripts lambdas/process_payment lambdas/confirm_booking .github/workflows
```

Cấu trúc mục tiêu:

```
court-booking-frontend/          # Danh & Hùng
├── src/
│   └── api/schema.d.ts          #   sinh từ OpenAPI của backend (§4.3)
├── package.json
├── amplify.yml                  # Build spec Amplify
└── .github/workflows/
    └── ci.yml                   # kiểm tra PR: tsc + eslint + vitest + contract check

court-booking-backend/           # Thành, Nguyên, Hiếu
├── backend/                     # FastAPI monolith
│   ├── app/                     #   mã ứng dụng
│   ├── alembic/                 #   migration DB
│   ├── deploy/
│   │   ├── appspec.yml          #   đặc tả CodeDeploy (copy vào gốc bundle khi deploy)
│   │   └── scripts/             #   các script lifecycle hook
│   ├── pyproject.toml
│   └── requirements.txt
├── lambdas/                     # Luồng thanh toán serverless
│   ├── process_payment/app.py
│   ├── confirm_booking/app.py
│   ├── template.yaml            #   SAM: 2 hàm + API GW + SNS
│   └── samconfig.toml
└── .github/workflows/
    ├── ci.yml                   # kiểm tra PR (lọc theo path: backend / lambdas)
    ├── deploy-backend.yml       # main → CodeDeploy → EC2 ASG
    └── deploy-lambdas.yml       # main → SAM deploy
```

#### 2.2 Bảo vệ nhánh (cả hai repo)

GitHub → từng repo → **Settings → Branches → Add rule** cho `main`:

- Yêu cầu pull request trước khi merge (1 phê duyệt)
- Yêu cầu status check pass — repo frontend: `ci / quality`, `ci / contract`; repo backend: `ci / backend`, `ci / lambdas` (chọn được sau lần chạy CI đầu tiên)
- Không cho phép bỏ qua các thiết lập trên

#### 2.3 GitHub Environments (cổng phê duyệt deploy — chỉ repo backend)

Cổng phê duyệt deploy nằm ở **repo backend** (frontend deploy qua Amplify, nơi merge vào `main` đã được bảo vệ bằng review PR):

GitHub → `court-booking-backend` → **Settings → Environments**:

1. Tạo `dev` — không quy tắc bảo vệ (tự động deploy khi merge).
2. Tạo `prod` — Required reviewers → thêm Hiếu. Mọi deploy prod sẽ chờ một cú nhấp phê duyệt.

> **Quy ước phối hợp giữa hai repo:** với tính năng liên quan cả hai phía, PR backend merge trước và deploy lên dev; PR frontend merge sau, khi contract check đã pass với API dev vừa cập nhật.

#### 2.4 Khởi tạo mã nguồn repo backend (từng bước)

Chạy các bước này trên máy của người khởi tạo repo (Hiếu). Thứ tự rất quan trọng: scaffold và push **trước khi** cấu hình bảo vệ nhánh (§2.2), để các status check tồn tại và chọn được.

**Bước 1 — chuẩn bị (một lần cho mỗi máy)**

```bash
git --version            # phiên bản mới bất kỳ
gh auth login            # GitHub CLI, đã cấp quyền cho <GH_ORG>
pyenv install 3.12       # một lần cho mỗi máy; khớp với runtime trên EC2 (§5.2)
```

> Nhóm thống nhất dùng **pyenv** cho Python trên máy local. CI (`actions/setup-python`) và EC2 (`dnf install python3.12`) đều pin cùng dòng 3.12, nên cả ba môi trường đồng nhất.

**Bước 2 — tạo repo, clone và dựng khung thư mục**

```bash
gh repo create <GH_ORG>/court-booking-backend --private --clone
cd court-booking-backend
pyenv local 3.12         # tạo file .python-version — hãy commit nó: đồng đội dùng pyenv sẽ tự chuyển phiên bản
mkdir -p backend/app/routers backend/tests backend/deploy/scripts \
         lambdas/process_payment lambdas/confirm_booking .github/workflows
```

**Bước 3 — dependency + virtualenv**

`backend/requirements.txt`:

```
fastapi
gunicorn
uvicorn[standard]
sqlalchemy>=2.0
psycopg[binary]
alembic
pydantic-settings
boto3
```

`backend/requirements-dev.txt`:

```
-r requirements.txt
pytest
httpx
ruff
```

Hai cách tạo môi trường tương đương — chọn cách nào cũng được; các phần phía sau không bị ảnh hưởng (CI tự tạo môi trường riêng, còn script deploy trên EC2 luôn dùng `/opt/court-booking/.venv` phía server):

**Cách A — pyenv-virtualenv** (tự động kích hoạt; env lưu tập trung trong `~/.pyenv/versions/`, không có thư mục thừa trong repo):

```bash
cd backend
pyenv virtualenv 3.12 fcaj-env
pyenv activate fcaj-env    # thêm eval "$(pyenv virtualenv-init -)" vào ~/.zshrc để tự kích hoạt theo thư mục
pip install -r requirements-dev.txt
```

> Giữ file `.python-version` được commit là `3.12` (bước 2). Nếu muốn tự kích hoạt bằng `pyenv local fcaj-env`, lưu ý lệnh này ghi *tên env* vào `.python-version` — chỉ commit khi cả nhóm cùng tạo env trùng tên.

**Cách B — venv thường** (không cần tool bổ sung; thư mục nằm trong repo và đã có trong `.gitignore`):

```bash
cd backend
python -m venv .venv && source .venv/bin/activate   # `python` trỏ tới 3.12 qua shim của pyenv
pip install -r requirements-dev.txt
```

> Editor tự nhận `.venv` theo quy ước; với env đặt tên riêng ở Cách A, chọn interpreter thủ công một lần (VS Code: `Python: Select Interpreter` → mục `fcaj-env` dưới pyenv).

**Bước 4 — ứng dụng FastAPI tối thiểu với endpoint `/health`**

Endpoint health được cố ý viết đầu tiên trong repo — nó vừa là đường health-check của ELB, vừa là đích kiểm tra của `validate.sh` trong CodeDeploy (§5.4).

`backend/app/__init__.py` — file rỗng.

`backend/app/config.py`:

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    app_env: str = "local"
    database_url: str = "postgresql+psycopg://app:app@localhost:5432/courtbooking_dev"


settings = Settings()
```

> Trên EC2, `DATABASE_URL` thật được nạp thành biến môi trường từ SSM Parameter Store (§3.5); `pydantic-settings` tự động đọc. Khi chạy local thì dùng giá trị mặc định ở trên.

`backend/app/main.py`:

```python
from fastapi import FastAPI

app = FastAPI(title="Court Booking API", version="0.1.0")


@app.get("/api/v1/health")
def health() -> dict:
    return {"status": "ok"}
```

`backend/tests/test_health.py`:

```python
from fastapi.testclient import TestClient

from app.main import app


def test_health():
    r = TestClient(app).get("/api/v1/health")
    assert r.status_code == 200
    assert r.json() == {"status": "ok"}
```

`backend/pyproject.toml` (cấu hình tooling):

```toml
[tool.ruff]
line-length = 100

[tool.pytest.ini_options]
pythonpath = ["."]
```

**Bước 5 — kiểm tra local trước mọi thứ khác**

```bash
uvicorn app.main:app --reload --port 8000   # từ backend/, đã kích hoạt venv
curl http://localhost:8000/api/v1/health    # → {"status":"ok"}
ruff check . && pytest                      # cả hai phải pass
```

**Bước 6 — Alembic + migration đầu tiên**

```bash
alembic init alembic
```

Trong `alembic/env.py` vừa sinh ra, cho URL đọc từ biến môi trường (thêm sau dòng `config = context.config`):

```python
import os

config.set_main_option(
    "sqlalchemy.url",
    os.environ.get("DATABASE_URL", "postgresql+psycopg://app:app@localhost:5432/courtbooking_dev"),
)
```

Để chạy migration local mà không cần cài Postgres trực tiếp vào máy, dùng Docker Compose chỉ cho phần database — cách này giống hệt service container mà CI đang dùng. `docker-compose.yml` ở gốc repo:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: courtbooking_dev
    ports: ["5432:5432"]
```

```bash
docker compose up -d db
```

Migration đầu tiên bật extension mà exclusion constraint chống đặt trùng sân cần dùng:

```bash
alembic revision -m "enable btree_gist"
```

```python
def upgrade():
    op.execute("CREATE EXTENSION IF NOT EXISTS btree_gist")


def downgrade():
    op.execute("DROP EXTENSION IF EXISTS btree_gist")
```

**Bước 7 — các file phục vụ deploy**

Chép các file từ §5.2/§5.4 vào: `backend/deploy/appspec.yml`, bốn script hook trong `backend/deploy/scripts/` và `backend/deploy/court-booking.service`. Sau đó:

```bash
chmod +x deploy/scripts/*.sh
```

**Bước 8 — khung Lambda**

`lambdas/process_payment/app.py` và `lambdas/confirm_booking/app.py`, mỗi file khởi đầu là:

```python
def handler(event, context):
    return {"statusCode": 200}
```

Kèm `lambdas/template.yaml` và `lambdas/samconfig.toml` từ §6.1, `lambdas/requirements-dev.txt` chứa `pytest`, và một smoke test `lambdas/test_handlers.py`:

```python
from process_payment.app import handler


def test_handler_returns_200():
    assert handler({}, None)["statusCode"] == 200
```

**Bước 9 — dọn dẹp repo + workflow**

`.gitignore` ở gốc repo:

```
.venv/
__pycache__/
*.pyc
.aws-sam/
.env
```

Thêm `.github/workflows/ci.yml` (bản backend ở §7). Khoan thêm `deploy-backend.yml`/`deploy-lambdas.yml` cho tới khi phía AWS (§3, §5, §6) sẵn sàng — một workflow deploy chỉ có thể fail thì chỉ gây nhiễu.

**Bước 10 — push lần đầu và kiểm chứng**

```bash
cd ..            # trở về gốc repo
git add -A
git commit -m "Scaffold backend repo: FastAPI skeleton, Alembic, deploy assets, SAM lambdas, CI"
git push -u origin main
```

**Kiểm chứng:** GitHub → Actions → run `ci` xanh: job `changes` nhận diện cả hai thành phần, job `backend` chạy ruff + alembic + pytest trên Postgres service container, job `lambdas` chạy smoke test.

**Bước 11 — cấu hình sau khi push**

1. Bảo vệ nhánh (§2.2) — lúc này check `ci / backend` và `ci / lambdas` đã chọn được.
2. Environments (§2.3).
3. Mời nhóm BE: repo → **Settings → Collaborators** → thêm Thành và Nguyên (quyền Write), hoặc:

```bash
gh api -X PUT repos/<GH_ORG>/court-booking-backend/collaborators/<username> -f permission=push
```

Từ đây, Thành & Nguyên tạo nhánh từ `main`, triển khai các router theo thiết kế API thống nhất ([Thiết kế kiến trúc §6.1](../2.1-architecture/)), và mỗi PR tự động đi qua toàn bộ quality gate.

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

`trust-backend.json` — chỉ workflow từ **repo backend** mới assume được. (Repo frontend hoàn toàn không cần AWS role: Amplify tự thực hiện deploy.)

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
          "token.actions.githubusercontent.com:sub": "repo:<GH_ORG>/court-booking-backend:*"
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

`amplify.yml` ở gốc repo `court-booking-frontend` (repo chuyên biệt nên không cần lớp bọc monorepo):

```yaml
version: 1
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

1. Console → **Amplify → Create new app → GitHub** → cấp quyền → chọn `<GH_ORG>/court-booking-frontend`, nhánh `main` (không có bước chọn app-root — gốc repo chính là ứng dụng).
2. **Environment variables**: `VITE_API_URL = https://<ELB_DNS_hoặc_domain>/api/v1`.
3. **App settings → Previews → Enable pull request previews** trên `main` — mỗi PR có URL riêng (vòng review của Danh & Hùng).
4. Định tuyến SPA — **Rewrites and redirects**, thêm rule 200 (Rewrite) từ
   `</^[^.]+$|\.(?!(css|js|map|json|png|svg|jpg|ico|txt|woff2?)$)([^.]+$)/>` về `/index.html`.

**Kiểm chứng:** push một commit lên repo frontend → Amplify console hiển thị build → URL hosting phục vụ ứng dụng. Mở PR → URL preview xuất hiện trong PR checks.

#### 4.3 Kiểm tra API contract (chi phí đánh đổi của hai repo — chỉ cần xử lý một lần)

Với hai repo tách biệt, thay đổi endpoint và phần UI gọi endpoint đó không còn nằm chung một PR — nên sai lệch giữa API do backend định nghĩa và kiểu dữ liệu phía frontend phải được phát hiện tự động. FastAPI vốn đã sinh sẵn schema tại `/openapi.json`; CI của frontend sinh lại kiểu dữ liệu client từ backend **dev** và fail nếu kết quả thay đổi mà chưa được commit:

```yaml
  contract:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - name: Pull backend OpenAPI schema (dev)
        run: curl -fsS https://<DEV_ELB_DNS>/openapi.json -o /tmp/openapi.json
      - name: Regenerate client types
        run: npx openapi-typescript /tmp/openapi.json -o src/api/schema.d.ts
      - name: Fail on uncommitted drift
        run: git diff --exit-code src/api/schema.d.ts
```

Khi backend thay đổi API, check `contract` của frontend sẽ fail ngay lúc mở PR — thay vì lỗi âm thầm lúc runtime. Cách xử lý cũng rất đơn giản: chạy lại `openapi-typescript` tại máy, commit file `schema.d.ts` mới sinh, và để `tsc` chỉ ra mọi chỗ trong UI cần cập nhật theo.

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

Mỗi repo mang `ci.yml` riêng. Bản của frontend cực kỳ đơn giản (không cần lọc path — cả repo là một ứng dụng); bản của backend chỉ giữ filter để tách hai thành phần nội bộ.

**`court-booking-frontend/.github/workflows/ci.yml`:**

```yaml
name: ci
on:
  pull_request:

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npx tsc --noEmit && npx eslint src && npx vitest run

  # contract: xem §4.3 — sinh lại kiểu từ /openapi.json của backend dev
```

**`court-booking-backend/.github/workflows/ci.yml`:**

```yaml
name: ci
on:
  pull_request:

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      backend: ${{ steps.f.outputs.backend }}
      lambdas: ${{ steps.f.outputs.lambdas }}
    steps:
      - uses: actions/checkout@v4
      - id: f
        uses: dorny/paths-filter@v3
        with:
          filters: |
            backend: ["backend/**"]
            lambdas: ["lambdas/**"]

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
| 1   | Tạo cả hai repo, bảo vệ nhánh, environments (§2)                 | Hiếu           | —                |
| 2   | OIDC provider + 2 deploy role + bucket artifact (§3)             | Hiếu           | Tài khoản AWS    |
| 3   | Tham số SSM (dev + prod)                                         | Hiếu           | Endpoint RDS     |
| 4   | Commit `ci.yml` ở cả hai repo — check hoạt động trước mọi mã     | Hiếu           | 1                |
| 5   | Kết nối Amplify với repo frontend + PR preview (§4)              | Danh & Hùng    | 1                |
| 6   | User-data launch template + instance role (§5.1–5.2)             | Hiếu           | VPC/ASG sẵn sàng |
| 7   | CodeDeploy app + deployment group (§5.3)                         | Hiếu           | 6                |
| 8   | `appspec.yml` + hooks + endpoint `/health` (§5.4)                | Thành & Nguyên | Khung FastAPI    |
| 9   | `deploy-backend.yml` chạy xanh lần đầu (§5.5)                    | Hiếu           | 7, 8             |
| 10  | Template SAM + `deploy-lambdas.yml` (§6)                         | Hiếu           | 2                |
| 11  | Job kiểm tra API contract (OpenAPI) trong repo frontend (§4.3)   | Danh & Hùng    | 5, 9             |
| 12  | Diễn tập rollback — thực hành từng dòng của §8 một lần           | cả nhóm        | 9, 10            |

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

---

### 11. Bảng thuật ngữ viết tắt

| Viết tắt | Ý nghĩa |
| --- | --- |
| API | Giao diện lập trình ứng dụng — quy ước để các thành phần phần mềm giao tiếp với nhau |
| ASG | Nhóm EC2 instance tự động tăng/giảm số lượng theo tải |
| AWS | Nền tảng điện toán đám mây của Amazon |
| BE | Backend — phần phía máy chủ của ứng dụng |
| CDK | Bộ công cụ định nghĩa hạ tầng AWS bằng ngôn ngữ lập trình |
| CI/CD | Tích hợp liên tục / Chuyển giao liên tục — tự động build, test và triển khai |
| DB | Cơ sở dữ liệu |
| EC2 | Amazon Elastic Compute Cloud — máy chủ ảo trên AWS |
| ECR | Kho lưu trữ image container được quản lý |
| ECS | Dịch vụ điều phối container của AWS |
| ELB | Bộ cân bằng tải — phân phối lưu lượng vào giữa các instance |
| FE | Frontend — phần giao diện phía người dùng của ứng dụng |
| GH | GitHub |
| GW | Gateway (API GW = Amazon API Gateway) |
| HMAC | Mã xác thực thông điệp dựa trên hàm băm — chứng minh tính toàn vẹn và nguồn gốc |
| HTTP | Giao thức truyền siêu văn bản |
| IaC | Hạ tầng dưới dạng mã — khai báo hạ tầng trong tệp có quản lý phiên bản |
| IAM | Quản lý danh tính và quyền truy cập của AWS — user, role và quyền hạn |
| NAT | Dịch địa chỉ mạng — cho phép subnet private đi ra internet |
| OIDC | Lớp danh tính xây trên OAuth 2.0 |
| PR | Pull Request — thay đổi mã được review trước khi merge |
| RDS | Dịch vụ cơ sở dữ liệu quan hệ được quản lý của AWS |
| S3 | Dịch vụ lưu trữ đối tượng của AWS |
| SAM | Framework hạ tầng-dưới-dạng-mã cho ứng dụng serverless của AWS |
| SG | Tường lửa ảo cấp instance |
| SNS | Dịch vụ thông báo pub/sub của AWS |
| SPA | Ứng dụng web một trang, định tuyến phía client |
| SSM | AWS Systems Manager — quản lý vận hành: tham số, chạy lệnh từ xa, phiên làm việc |
| TS | TypeScript — JavaScript có kiểu tĩnh |
| UI | Giao diện người dùng |
| URL | Địa chỉ tài nguyên trên web |
| VPC | Mạng ảo riêng biệt trên AWS |
| YAML | Định dạng cấu hình dễ đọc cho con người |
