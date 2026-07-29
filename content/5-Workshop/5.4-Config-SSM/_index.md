---
title: "Configuration & Secrets (SSM)"
date: 2026-07-30
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

No configuration lives in the repo, in GitHub, or baked into an AMI. **SSM Parameter Store is the single source of runtime config**, under one path per environment:

```
/court-booking/dev/APP_ENV                 String        dev
/court-booking/dev/DATABASE_URL            SecureString  postgresql+psycopg://app:…@<rds-endpoint>:5432/courtbooking
/court-booking/dev/COGNITO_USER_POOL_ID    String        (from section 5.5)
/court-booking/dev/COGNITO_CLIENT_ID       String        (from section 5.5)
/court-booking/dev/CORS_ORIGINS            String        ["https://<amplify-url>"]
```

Secrets (the DB URL, which embeds the password) are **SecureString** — encrypted at rest with KMS; plain identifiers are `String`.

![Add SSM parameter](/images/5-Workshop/5.4/ssm_add_param_0.png)

![Add SSM parameter — SecureString](/images/5-Workshop/5.4/ssm_add_param_1.png)

#### The bridge: how parameters reach the application

The FastAPI app reads settings from **environment variables** (pydantic `BaseSettings` in `config.py`) — it does not call SSM itself. The link is made at **deploy time**: the CodeDeploy `install` hook pulls every parameter under the path and materialises an env file that systemd feeds to the process:

```bash
# deploy/scripts/install.sh (runs on the EC2 instance at every deploy)
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

The parameter's **leaf name** (`DATABASE_URL`) becomes the env var name, which pydantic matches to the `database_url` settings field. Changing config = update the parameter + redeploy (or restart); no code, no image rebuild.

{{% notice note %}}
Two details that matter: the EC2 **instance role** needs `ssm:GetParameter*` on the path plus `kms:Decrypt` (for SecureString); and `CORS_ORIGINS` is a *list* field in pydantic, so its value must be a **JSON array string** (`["https://…"]`) — a bare URL fails to parse.
{{% /notice %}}

{{% notice tip %}}
Why not GitHub Secrets? Those exist only inside workflow runs. SSM parameters are readable by the *instance* at boot/deploy — which also makes them Auto-Scaling-safe: a brand-new instance launched by the ASG self-configures with zero human action.
{{% /notice %}}
