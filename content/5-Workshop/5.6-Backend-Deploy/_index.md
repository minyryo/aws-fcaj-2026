---
title: "Backend Deploy (EC2 + CodeDeploy)"
date: 2026-07-30
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

Deploying the backend is **one action: `git push`**. Everything else is automation. This section shows the chain and the assets that make it work.

#### 1. The deploy chain

```
git push (backend/**)                                 developer
  → deploy-backend.yml                                GitHub Actions
  → OIDC token → AssumeRole gh-deploy-backend         no AWS keys (5.1)
  → zip revision → upload to deploy S3 bucket
  → alembic upgrade head via SSM Run Command          (DB migration, gated — see §4)
  → aws deploy create-deployment                      CodeDeploy
      → agent on EC2 pulls the zip from S3
      → lifecycle hooks run (appspec.yml):
          ApplicationStop   deploy/scripts/stop.sh      systemctl stop
          AfterInstall      deploy/scripts/install.sh   venv + pip install,
                                                        SSM → /etc/court-booking.env (5.4),
                                                        install systemd unit
          ApplicationStart  deploy/scripts/start.sh     systemctl start (gunicorn :8000)
          ValidateService   deploy/scripts/validate.sh  poll /api/v1/health until 200
  → ALB target-group health check passes → instance serves traffic
```

The instance never holds credentials or config in its AMI: **code arrives from S3, config arrives from SSM** — both at deploy time. That is what makes the setup Auto-Scaling-safe: a fresh instance launched by the ASG runs the CodeDeploy agent, pulls the latest revision, and self-configures.

#### 2. Instance prerequisites (launch-template user-data)

The hooks assume four things exist on the box, provisioned once in user-data: `python3.12`, a non-login `fastapi` system user, the **CodeDeploy agent**, and `/opt/court-booking`. The instance profile grants only SSM parameter reads (+ `kms:Decrypt`) and S3 read on the deploy bucket.

#### 3. The deploy assets (committed in `backend/deploy/`)

- `appspec.yml` — maps the bundle to `/opt/court-booking` and binds the four hooks
- `scripts/stop.sh` — `systemctl stop court-booking || true` (tolerates the very first deploy, when the service doesn't exist yet)
- `scripts/install.sh` — venv + `pip install -r requirements.txt`, the SSM→env bridge, systemd unit install
- `scripts/start.sh` / `scripts/validate.sh` — start, then poll `localhost:8000/api/v1/health` (24 × 5 s) before CodeDeploy declares success
- `court-booking.service` — gunicorn with the uvicorn worker, `User=fastapi`, `EnvironmentFile=/etc/court-booking.env`, `Restart=always`

{{% notice warning %}}
Two mistakes we caught **before** they burned a deploy: (1) hook scripts were committed without the execute bit — CodeDeploy runs them directly and fails with `permission denied`; fixed with `git update-index --chmod=+x deploy/scripts/*.sh`. (2) `install.sh` copied a systemd unit that wasn't in the repo yet — the `AfterInstall` hook would have died mid-deploy.
{{% /notice %}}

#### 4. First deploy: the walking skeleton

There is a chicken-and-egg problem: the workflow migrates the DB **via SSM before** CodeDeploy lands the code — but on the very first deploy there is no venv on the box yet. We sequence around it:

1. **Deploy with `APP_ENV=local`** — the app boots without touching RDS; `/api/v1/health` returns 200. This one deploy proves the entire chain (OIDC → S3 → CodeDeploy → hooks → systemd → ALB) while the stakes are trivial.
2. **Bootstrap the DB from inside the VPC** — now the box has code + venv: run `alembic upgrade head` (schema + `btree_gist`) and load the seed via SSM Run Command. The DB is not publicly accessible, so this is the *only* path to it.
3. **Flip `APP_ENV` to `dev`** in SSM and redeploy — the app now runs against RDS + Cognito. Every subsequent push migrates automatically.

{{% notice tip %}}
The **walking-skeleton principle** ([v2 guide] §7): deploy the thinnest possible thing end-to-end first, then add substance to a working pipeline. Debugging IAM, CodeDeploy and systemd at the same time as database connectivity is strictly harder than debugging them one at a time.
{{% /notice %}}
