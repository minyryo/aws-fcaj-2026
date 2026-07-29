---
title: "Prerequisites & Identities"
date: 2026-07-30
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

Before any resource is created, we set up **who is allowed to do what**. Three identities exist, and everything later in this workshop runs as one of them:

| Identity | Kind | Used for |
| --- | --- | --- |
| `hieu-cli` (+ MFA) | IAM user, Hieu's account | the *human* setup credential |
| `court-booking-admin` | IAM role, workload account | cross-account AssumeRole target for all infra work |
| `gh-deploy-backend` / `gh-deploy-lambdas` | IAM roles, workload account | *keyless* CI deploys via GitHub OIDC |

The detailed policies live in the repo guide (`cicd-setup-guide.md` §1.0–§1.3); this page shows the result and the problems we hit.

#### 1. Human identity: cross-account AssumeRole with MFA

The workload account is Thanh's, but infrastructure is administered from Hieu's IAM user. Instead of sharing access keys, Thanh's account exposes a role `court-booking-admin` that trusts Hieu's user **only with MFA present**:

![Switch role setup](/images/5-Workshop/5.1/iam-switch_role-setup.png)

In the console this appears as **Switch role**, and on the CLI as a profile that chains `source_profile` + `role_arn` + `mfa_serial`:

```ini
# ~/.aws/config
[profile thanh]
role_arn       = arn:aws:iam::<WORKLOAD_ACCOUNT>:role/court-booking-admin
source_profile = hieu
mfa_serial     = arn:aws:iam::<HIEU_ACCOUNT>:mfa/hieu-cli-ip16pm
```

![Switch role demo](/images/5-Workshop/5.1/iam-switch_role-demo.png)

Verification — `get-caller-identity` must show the **assumed role**, not the IAM user:

![Verify AssumeRole](/images/5-Workshop/5.1/iam-verify_assumeRole.png)

{{% notice warning %}}
**Error we hit: MFA device name mismatch.** The CLI kept rejecting MFA codes because `mfa_serial` in `~/.aws/config` said `.../mfa/hieu-cli` while the actual device was registered as `.../mfa/hieu-cli-ip16pm`. The MFA serial is an exact ARN — verify the identifier in the IAM console and copy it verbatim.
{{% /notice %}}

![MFA wrong device name](/images/5-Workshop/5.1/error-iam-mfa-wrong_device_name.png)

Fix — check the real device identifier, then update the config:

![Verify MFA identifier](/images/5-Workshop/5.1/error-iam-mfa-wrong_device_name-solution-verify_identifier.png)

![Update aws config](/images/5-Workshop/5.1/error-iam-mfa-wrong_device_name-solution-update_aws_config.png)

#### 2. The admin role's permissions

The first policy attempt was too narrow, producing a chain of `AccessDenied` ("permission whack-a-mole") — each new AWS service needed another statement:

![Not authorized error](/images/5-Workshop/5.1/error-iam-not_authorized.png)

For a *setup* role (not a runtime role) we settled on broad admin permissions — this identity exists to build the infrastructure, is MFA-gated, and is not used by any pipeline:

![Admin role policy](/images/5-Workshop/5.1/iam-policy_court-booking-admin.png)

#### 3. CI identities: GitHub OIDC (no AWS keys in GitHub)

Deploy pipelines never hold access keys. GitHub Actions exchanges its **OIDC token** for short-lived credentials by assuming a role whose trust policy pins the exact repo and branch:

![gh-deploy-backend role](/images/5-Workshop/5.1/iam_role_gh-deploy-backend.png)

![gh-deploy-lambdas role](/images/5-Workshop/5.1/iam_role_gh-deploy-lambdas.png)

The backend role can only: upload to the deploy bucket, trigger CodeDeploy, and run the SSM migration command. The lambda role is scoped to SAM/CloudFormation. Least privilege here is real — these roles run unattended on every push.

{{% notice note %}}
**Secret hygiene**, enforced throughout the project: no access keys in any repo (OIDC removes the need), runtime secrets only in SSM Parameter Store (section 5.4), and the local `infra/.env` holding the DB password is gitignored.
{{% /notice %}}
