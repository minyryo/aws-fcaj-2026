---
title: "Deployment Process (CI/CD)"
date: 2026-07-11
weight: 2
chapter: false
pre: " <b> 2.2. </b> "
---

# CI/CD — Design & Step-by-Step Setup Guide

Summary:

- **Owner:** Hieu (Backend — Repo & Deployment, assigned 07/05/2026)
- **Audience:** the whole team (FE: Danh, Hung · BE: Thanh, Nguyen, Hieu)
- **Stack:** React + Vite (TS) on Amplify · FastAPI (Python) on EC2/ASG · Lambda + SAM · RDS PostgreSQL

### 1. CI/CD Design Overview

**Design principle:** GitHub Actions as the single orchestrator; AWS-native services as deploy targets. Two repositories split along team ownership — `court-booking-frontend` (Danh & Hung) and `court-booking-backend` (FastAPI + Lambdas, BE team) — with three deployment paths matching the three architecture zones:

```
              ┌─ court-booking-frontend ─┐   ┌────── court-booking-backend ──────┐
              │    React + Vite (TS)     │   │   /backend           /lambdas     │
              └────────────┬─────────────┘   └───────┬──────────────────┬────────┘
PR opened:      lint + test + contract        lint + test          lint + test
                       check                   (path-filtered within the repo)
merge to main:             ▼                          ▼                  ▼
              ┌────────────────┐             ┌─────────────┐    ┌──────────────┐
              │    Amplify     │             │ GH Actions  │    │ GH Actions   │
              │ built-in CI/CD │             │ → CodeDeploy│    │ → SAM deploy │
              └────────────────┘             │ → EC2 ASG   │    │ → Lambda ×2  │
                                             └─────────────┘    │ + API GW     │
                                                                └──────────────┘
```

| Component                       | Pipeline                                              | Rationale                                                                                                                                                                       |
| ------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Frontend** (React + Vite)     | Amplify Hosting built-in CI/CD                        | Already in the architecture diagram; free build minutes, automatic **PR preview URLs** for UI review, one-click rollback, zero YAML to maintain.                                |
| **Backend** (FastAPI on EC2)    | GitHub Actions → AWS CodeDeploy → ASG rolling deploy  | CodeDeploy is **Auto-Scaling-aware**: instances launched later by the ASG automatically receive the latest revision. ELB-health-gated rollout = zero downtime with 2 instances. |
| **Lambdas** (payment + confirm) | GitHub Actions → AWS SAM                              | Both functions + API Gateway + SNS wiring as reviewable infrastructure-as-code in one `template.yaml`.                                                                          |
| **DB migrations** (Alembic)     | Pipeline step before backend deploy, run once via SSM | Prevents schema drift; the exclusion constraint lives in a versioned migration.                                                                                                 |

**Cross-cutting decisions:**

1. **GitHub OIDC → IAM roles** — no long-lived AWS keys in GitHub secrets; one least-privilege role per pipeline.
2. **Two repos along team ownership** — FE and BE ship on independent cadences; API contract drift between them is caught by an OpenAPI type-check job in the frontend repo (§4.3). Inside the backend repo, path filters keep the backend and lambda pipelines independent.
3. **Two environments, gated** — merges auto-deploy to dev; prod waits behind a GitHub Environment approval.
4. **Quality gates on every PR** — `ruff` + `pytest` (+ Postgres service container), `tsc` + `eslint` + `vitest`.

#### Alternatives considered — and why they weren't picked

**Full AWS CodePipeline + CodeBuild** end-to-end would be the "pure AWS" answer, but: CodeBuild costs money per build minute where GitHub Actions is free for public/edu repos, the developer experience is clunkier (log diving in the console vs. PR-inline checks), and the team already lives in GitHub. We still get real AWS-service depth via CodeDeploy + SAM + Amplify + OIDC/IAM — arguably the best of both worlds, and each of those is a résumé-relevant AWS service.

**A single monorepo instead of two repos.** One repo gives atomic cross-stack PRs and a single review flow — attractive on paper. But the FE pair and the BE trio barely touch each other's code, the toolchains are disjoint (npm vs. Python), and Amplify connects more simply to a dedicated frontend repo. The one monorepo advantage that genuinely matters — keeping the API contract in sync — is recovered by the OpenAPI type-generation check (§4.3), so the repo boundary follows the team boundary instead.

**Containers on ECS/Fargate instead of EC2 + CodeDeploy.** Containerizing FastAPI would modernize the deploy story (immutable images, local/prod parity), but it replaces the EC2 monolith committed to in the [Architecture Design](../2.1-architecture/) — a different compute architecture, not just a different pipeline. It also adds Docker, ECR, and ECS as three new things to learn inside an 8-week window, for no functional gain at this scale. Worth revisiting after the program if the app lives on.

**CDK or Terraform instead of SAM for the serverless path.** Terraform brings third-party state management, which cuts against the "maximise AWS services" principle. CDK is excellent but is a general-purpose IaC framework — heavier to learn than SAM's declarative template for what is exactly two functions, one HTTP API, and one SNS topic. SAM is the smallest tool that does the job, and a SAM → CDK migration later is straightforward.

**GitHub Actions → S3 + CloudFront for the frontend instead of Amplify.** More control (cache policies, invalidations), but everything Amplify gives for free would have to be hand-rolled — most importantly per-PR preview URLs, which is the FE team's entire review loop.

#### Prerequisites and constraints

1. **CodeDeploy's agent setup on EC2 has a learning curve** (install the agent in launch-template user-data, `appspec.yml` hooks). Budget half a day for it. The fallback if it fights you: GitHub Actions → SSM `RunCommand` with a deploy script — simpler, but it loses ASG-awareness and native rollback, so treat it as plan B only.
2. **Lambda-in-VPC needs a route to the internet.** The process-payment function sits in private subnets to reach RDS — but if it must _call out_ to the payment gateway's API (verify signatures, query transaction status), private subnets have no internet route. Options: a NAT gateway (~$32/month — nearly doubles the project budget), a NAT instance on a t3.micro (free-tier-friendly, more ops work), or designing the webhook flow so the Lambda never initiates outbound calls (authenticate the incoming payload via HMAC signature instead). Decide this **before** wiring the subnets/SGs in the SAM template.
3. **CloudFormation deploys are not instant.** `sam deploy` runs a changeset — expect 1–3 minutes even for a one-line Lambda change. That's normal; don't chase it.
4. **The migration step assumes at least one healthy instance is running.** The SSM step targets the first running tagged instance — fine at this scale, but it's an assumption worth knowing when debugging a first-ever deploy into an empty ASG.

> Placeholders used throughout — replace globally before running anything:
> `<ACCOUNT_ID>` (12-digit AWS account), `<REGION>` (e.g. `ap-southeast-1`), `<GH_ORG>` (GitHub org or username). The two repos are named `court-booking-frontend` and `court-booking-backend`.

---

### 2. Repository Initialization

#### 2.1 Create the two repositories

The repo boundary follows the team boundary: the FE pair owns one repo, the BE trio owns the other. The Lambdas live **with the backend** — they share the database schema, the Alembic migration history, and the same owners.

```bash
gh repo create <GH_ORG>/court-booking-frontend --private --clone
gh repo create <GH_ORG>/court-booking-backend  --private --clone

cd court-booking-frontend
mkdir -p src .github/workflows

cd ../court-booking-backend
mkdir -p backend/app backend/alembic backend/deploy/scripts lambdas/process_payment lambdas/confirm_booking .github/workflows
```

Target structures:

```
court-booking-frontend/          # Danh & Hung
├── src/
│   └── api/schema.d.ts          #   generated from the backend's OpenAPI (§4.3)
├── package.json
├── amplify.yml                  # Amplify build spec
└── .github/workflows/
    └── ci.yml                   # PR checks: tsc + eslint + vitest + contract check

court-booking-backend/           # Thanh, Nguyen, Hieu
├── backend/                     # FastAPI monolith
│   ├── app/                     #   application code
│   ├── alembic/                 #   DB migrations
│   ├── deploy/
│   │   ├── appspec.yml          #   CodeDeploy spec (copied to bundle root at deploy)
│   │   └── scripts/             #   lifecycle hook scripts
│   ├── pyproject.toml
│   └── requirements.txt
├── lambdas/                     # Serverless payment path
│   ├── process_payment/app.py
│   ├── confirm_booking/app.py
│   ├── template.yaml            #   SAM: 2 functions + API GW + SNS
│   └── samconfig.toml
└── .github/workflows/
    ├── ci.yml                   # PR checks (path-filtered: backend / lambdas)
    ├── deploy-backend.yml       # main → CodeDeploy → EC2 ASG
    └── deploy-lambdas.yml       # main → SAM deploy
```

#### 2.2 Branch protection (both repos)

GitHub → each repo → **Settings → Branches → Add rule** for `main`:

- Require a pull request before merging (1 approval)
- Require status checks to pass — frontend repo: `ci / quality`, `ci / contract`; backend repo: `ci / backend`, `ci / lambdas` (selectable after the first CI run)
- Do not allow bypassing the above settings

#### 2.3 GitHub Environments (deploy gate — backend repo only)

The deploy gates live in the **backend repo** (the frontend deploys via Amplify, where merge-to-main is already protected by PR review):

GitHub → `court-booking-backend` → **Settings → Environments**:

1. Create `dev` — no protection rules (auto-deploy on merge).
2. Create `prod` — Required reviewers → add Hieu. Every prod deploy then waits for one click of approval.

> **Cross-repo convention:** for features touching both sides, the backend PR merges first and reaches dev; the frontend PR follows once the contract check passes against the updated dev API.

---

### 3. AWS One-Time Preparation

#### 3.1 OIDC identity provider (no AWS keys in GitHub!)

GitHub Actions will assume IAM roles via OpenID Connect. Create the provider once:

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

#### 3.2 Deploy role — backend pipeline

`trust-backend.json` — only workflows from the **backend repo** can assume it. (The frontend repo needs no AWS role at all: Amplify performs its own deployments.)

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

Permissions policy (least privilege — S3 revision bucket + CodeDeploy + SSM for migrations):

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

#### 3.3 Deploy role — lambda pipeline

Same trust policy → role `gh-deploy-lambdas`. Attach permissions for SAM/CloudFormation deploys (CloudFormation on stack `court-booking-payment*`, `iam:PassRole` for the SAM-created function roles, S3 for the SAM artifact bucket, and Lambda/APIGW/SNS write actions). For the 8-week project scope, `PowerUserAccess` scoped by a permissions boundary is an acceptable shortcut — tighten later.

#### 3.4 Artifact bucket

```bash
aws s3 mb s3://court-booking-deploy-<ACCOUNT_ID> --region <REGION>
aws s3api put-bucket-versioning --bucket court-booking-deploy-<ACCOUNT_ID> \
  --versioning-configuration Status=Enabled
```

#### 3.5 Runtime secrets in SSM Parameter Store

Never put DB credentials in GitHub. The app reads them at startup from SSM:

```bash
aws ssm put-parameter --name /court-booking/dev/DATABASE_URL \
  --type SecureString --value "postgresql://app:<PASSWORD>@<RDS_ENDPOINT>:5432/courtbooking_dev"
aws ssm put-parameter --name /court-booking/prod/DATABASE_URL \
  --type SecureString --value "postgresql://app:<PASSWORD>@<RDS_ENDPOINT>:5432/courtbooking"
# likewise: /court-booking/<env>/COGNITO_USER_POOL_ID, /COGNITO_CLIENT_ID, /SNS_TOPIC_ARN ...
```

---

### 4. Frontend — Amplify Hosting (built-in CI/CD)

#### 4.1 Build spec

`amplify.yml` at the root of `court-booking-frontend` (a dedicated repo means no monorepo wrapper is needed):

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

#### 4.2 Connect the repo

1. Console → **Amplify → Create new app → GitHub** → authorize → select `<GH_ORG>/court-booking-frontend`, branch `main` (no app-root prompt — the repo root *is* the app).
2. **Environment variables**: `VITE_API_URL = https://<ELB_DNS_or_domain>/api/v1`.
3. **App settings → Previews → Enable pull request previews** on `main` — every PR gets its own URL (this is Danh & Hung's review loop).
4. SPA routing — **Rewrites and redirects**, add a 200 (Rewrite) rule from
   `</^[^.]+$|\.(?!(css|js|map|json|png|svg|jpg|ico|txt|woff2?)$)([^.]+$)/>` to `/index.html`.

**Verify:** push a commit to the frontend repo → Amplify console shows the build → the hosted URL serves the app. Open a PR → a preview URL appears in the PR checks.

#### 4.3 API contract check (the price of two repos, paid once)

With separate repos, an endpoint change and its UI usage no longer land in one PR — so drift between the FastAPI contract and the frontend types must be caught mechanically. FastAPI already publishes its schema at `/openapi.json`; the frontend CI regenerates its client types from the **dev** backend and fails if they changed without being committed:

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

A backend contract change then breaks the frontend's `contract` check loudly at PR time — instead of silently at runtime. When it fires, the fix is one command locally: rerun `openapi-typescript`, commit the regenerated `schema.d.ts`, and let `tsc` point at every UI usage that needs updating.

---

### 5. Backend — EC2/ASG via CodeDeploy

#### 5.1 EC2 instance role

The instances need to _pull_ revisions and _read_ SSM parameters. Create role `court-booking-ec2` (EC2 trust) with:

- `AmazonSSMManagedInstanceCore` (Session Manager + Run Command — also used for migrations)
- Inline: `s3:GetObject` on `arn:aws:s3:::court-booking-deploy-<ACCOUNT_ID>/*`
- Inline: `ssm:GetParameter*` on `arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/court-booking/*` + `kms:Decrypt` on the default SSM key

Attach as **instance profile** in the launch template.

#### 5.2 Launch template user-data (CodeDeploy agent + app runtime)

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

> The agent polls CodeDeploy on boot — this is what makes deployments **Auto-Scaling-safe**: any instance the ASG launches later automatically installs the latest revision before entering service.

systemd unit `/etc/systemd/system/court-booking.service` (installed by the deploy hook below):

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

# service role for CodeDeploy itself
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

`OneAtATime` + target-group integration = rolling deploy: each instance is deregistered from the ELB, updated, health-checked, re-registered — **zero downtime with 2 instances**.

#### 5.4 appspec.yml + hook scripts

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

> `validate.sh` failing ⇒ CodeDeploy marks the deployment failed and (with rollback enabled) reverts to the previous revision automatically. Implement `GET /api/v1/health` in FastAPI on day one — it's also the ELB health-check path.

#### 5.5 GitHub Actions workflow

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

concurrency: deploy-backend # never two deploys at once

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
    environment: prod # waits for approval (section 2.3)
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

      - name: Run DB migrations (once, via SSM on one instance)
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

> **Migration ordering rule:** migrations run _before_ the new code deploys, so every migration must be **backwards-compatible with the currently running code** (add columns as nullable, never drop-and-rename in one release). This is the only discipline the team must learn about zero-downtime deploys.

**Verify:** merge a trivial change under `backend/` → Actions runs tests → waits at the `prod` gate → approve → CodeDeploy console shows the OneAtATime rollout → `curl https://<ELB_DNS>/api/v1/health` returns 200.

---

### 6. Lambdas — SAM Pipeline

#### 6.1 SAM template

`lambdas/template.yaml` (skeleton — the two functions from the architecture, VPC-attached to reach RDS):

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
      SecurityGroupIds: [!Ref LambdaSG] # SG allowed into the RDS SG on 5432
      SubnetIds: [subnet-xxxx, subnet-yyyy] # private subnets
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

**Verify:** merge a change under `lambdas/` → stack `court-booking-payment-prod` updates in CloudFormation → `curl -X POST <WebhookUrl>` with a test payload → check the function's CloudWatch log group.

---

### 7. PR Checks (the quality gate)

Each repo carries its own `ci.yml`. The frontend one is trivially simple (no path filtering needed — the whole repo is one app); the backend one keeps a filter only to separate its two internal components.

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

  # contract: see §4.3 — regenerates types from the dev backend's /openapi.json
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
          alembic upgrade head      # migrations are tested on every PR too
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

> The PR Postgres service container means **the exclusion constraint and row-locking logic are exercised on every PR** — the double-booking defense is continuously tested.

---

### 8. Rollback Runbook

| Component      | How to roll back                                                                                                                                                                | Time         |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| Frontend       | Amplify console → app → **Redeploy** a previous successful build                                                                                                                | ~2 min       |
| Backend        | CodeDeploy console → deployments → previous revision → **Retry/redeploy**; or `aws deploy create-deployment` pointing at the previous `backend-<sha>.zip` (bucket is versioned) | ~5 min       |
| Backend (auto) | `validate.sh` failure triggers automatic rollback if enabled: `aws deploy update-deployment-group ... --auto-rollback-configuration enabled=true,events=DEPLOYMENT_FAILURE`     | automatic    |
| Lambdas        | `git revert` the merge → pipeline redeploys previous template/code; or CloudFormation console → stack → previous change set                                                     | ~5 min       |
| DB migration   | `alembic downgrade -1` via SSM Run Command — **only if the migration is reversible**; prefer roll-forward fixes                                                                 | case-by-case |

---

### 9. Setup Order & Ownership Checklist

| #   | Task                                                  | Owner          | Depends on       |
| --- | ----------------------------------------------------- | -------------- | ---------------- |
| 1   | Create both repos, branch protection, environments (§2)   | Hieu           | —                |
| 2   | OIDC provider + 2 deploy roles + artifact bucket (§3)      | Hieu           | AWS account      |
| 3   | SSM parameters (dev + prod)                                | Hieu           | RDS endpoint     |
| 4   | Commit `ci.yml` in both repos — checks live before app code | Hieu           | 1                |
| 5   | Connect Amplify to the frontend repo + PR previews (§4)    | Danh & Hung    | 1                |
| 6   | Launch template user-data + instance role (§5.1–5.2)       | Hieu           | VPC/ASG exists   |
| 7   | CodeDeploy app + deployment group (§5.3)                   | Hieu           | 6                |
| 8   | `appspec.yml` + hooks + `/health` endpoint (§5.4)          | Thanh & Nguyen | FastAPI skeleton |
| 9   | `deploy-backend.yml` first green run (§5.5)                | Hieu           | 7, 8             |
| 10  | SAM template + `deploy-lambdas.yml` (§6)                   | Hieu           | 2                |
| 11  | OpenAPI contract-check job in the frontend repo (§4.3)     | Danh & Hung    | 5, 9             |
| 12  | Rollback drill — practice each row of §8 once              | everyone       | 9, 10            |

---

### 10. Cost Impact

| Service                        | Added cost                                                      |
| ------------------------------ | --------------------------------------------------------------- |
| GitHub Actions                 | $0 (free tier: 2,000 min/month private repos; unlimited public) |
| Amplify Hosting build          | free tier 1,000 build-min/month, then ~$0.01/min                |
| CodeDeploy (EC2)               | **$0** — free for EC2/on-prem                                   |
| SAM / CloudFormation           | $0 (pay only for the Lambdas themselves)                        |
| S3 artifact bucket             | < $0.10/month                                                   |
| SSM Parameter Store (standard) | $0                                                              |

Net: effectively **zero** on top of the existing ~$45/month runtime estimate from the [Architecture Design](../2.1-architecture/).

---

### 11. Glossary

| Abbreviation | Meaning                                                                                        |
| ------------ | ---------------------------------------------------------------------------------------------- |
| API          | Application Programming Interface — the contract through which software components communicate |
| ASG          | Auto Scaling Group — a group of EC2 instances that automatically scales in/out with load       |
| AWS          | Amazon Web Services — Amazon's cloud computing platform                                        |
| BE           | Backend — the server-side part of the application                                              |
| CDK          | AWS Cloud Development Kit — define AWS infrastructure in general-purpose programming languages |
| CI/CD        | Continuous Integration / Continuous Delivery — automated building, testing, and deployment     |
| DB           | Database                                                                                       |
| EC2          | Amazon Elastic Compute Cloud — virtual servers on AWS                                          |
| ECR          | Amazon Elastic Container Registry — managed Docker image registry                              |
| ECS          | Amazon Elastic Container Service — container orchestration service                             |
| ELB          | Elastic Load Balancer — distributes incoming traffic across instances                          |
| FE           | Frontend — the client-side part of the application                                             |
| GH           | GitHub                                                                                         |
| GW           | Gateway (API GW = Amazon API Gateway)                                                          |
| HMAC         | Hash-based Message Authentication Code — signature proving a message's integrity and origin    |
| HTTP         | HyperText Transfer Protocol                                                                    |
| IaC          | Infrastructure as Code — declaring infrastructure in versioned files                           |
| IAM          | AWS Identity and Access Management — users, roles, and permissions                             |
| NAT          | Network Address Translation — lets private subnets reach the internet                          |
| OIDC         | OpenID Connect — identity layer on top of OAuth 2.0                                            |
| PR           | Pull Request — a proposed code change reviewed before merging                                  |
| RDS          | Amazon Relational Database Service — managed SQL databases                                     |
| S3           | Amazon Simple Storage Service — object storage                                                 |
| SAM          | AWS Serverless Application Model — IaC framework for Lambda applications                       |
| SG           | Security Group — instance-level virtual firewall                                               |
| SNS          | Amazon Simple Notification Service — pub/sub messaging and notifications                       |
| SPA          | Single-Page Application — web app served as one HTML page with client-side routing             |
| SSM          | AWS Systems Manager — fleet management: parameters, run-command, sessions                      |
| TS           | TypeScript — statically typed JavaScript                                                       |
| UI           | User Interface                                                                                 |
| URL          | Uniform Resource Locator — web address                                                         |
| VPC          | Amazon Virtual Private Cloud — an isolated virtual network on AWS                              |
| YAML         | YAML Ain't Markup Language — human-readable configuration format                               |
