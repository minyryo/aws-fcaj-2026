---
title: "Week 7 Worklog"
date: 2026-07-27
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

> **Current week** (as of 07/31/2026). Items still in progress are marked accordingly.

### Week 7 Objectives:

- Provision the **AWS infrastructure**: network (VPC security-group chain), RDS PostgreSQL, Cognito user pool, SSM Parameter Store config.
- **Deploy the backend to EC2** through the CI/CD pipeline.
- Set up **Amplify hosting + custom domain** for the frontend.
- Revise the **architecture diagram** per mentor feedback and re-align the Proposal.

### Tasks to be carried out this week:

| Day | Task | Start Date | Completion Date | Reference Material |
| --- | ---- | ---------- | --------------- | ------------------ |
| 3 | **Amplify hosting**: connect the frontend repo, build env vars (`VITE_API_BASE_URL`, `VITE_USE_MOCK_API=false`), PR previews; custom domain `app.fcaj-rally.minervaph.works` | 07/28/2026 | 07/28/2026 | [Workshop §5.7](/5-workshop/5.7-frontend-amplify/) |
| 4 | **Network + data**: SG chain (ALB → app → RDS) + DB subnet group; **RDS PostgreSQL 16**; **Cognito** user pool + app client + role groups; **SSM parameters** (`DATABASE_URL`, `COGNITO_*`, `S3_BUCKET`, `CORS_ORIGINS`) | 07/29/2026 | 07/29/2026 | [Workshop §5.3–5.5](/5-workshop/) |
| 5 | **Compute + deploy path**: EC2 instance role, launch template, ASG, ALB; CodeDeploy blocked by the account's Free Tier plan (`SubscriptionRequiredException`) → adopted the **SSM Run Command** deploy path. **Architecture diagram v6** redrawn per mentor feedback (VPC/subnets, NAT, Multi-AZ, numbered flow) | 07/30/2026 | 07/30/2026 | [Workshop §5.6](/5-workshop/5.6-backend-deploy/) |
| 6 | **Deployment debugging** (OIDC trust-policy repo mismatch, unset `AWS_ACCOUNT_ID` var, missing `court-booking.service`, migration-ordering `127`); **Proposal §2.1 re-aligned to the 22-step diagram**; began the Workshop (§5) deployment runbook | 07/31/2026 | *In progress* | [Proposal §2.1](/2-proposal/2.1-architecture/) |
| 7–CN | **Backend deploy green + FE↔API wiring**; team meeting (08/02/2026) | 08/01/2026 | *Planned* | |

### Week 7 Achievements:

- **Frontend live on Amplify** at `https://app.fcaj-rally.minervaph.works` — git-connected CI/CD, build env vars, custom domain.
- **Core AWS infrastructure provisioned** — VPC security-group chain, RDS PostgreSQL 16, Cognito user pool + groups, and SSM Parameter Store as the single source of runtime config. Several Free-Tier-plan constraints were navigated (backup-retention and Multi-AZ restrictions, RDS master-password character rules).
- **Deploy pipeline built and debugged** — because the account's Free Tier plan blocks CodeDeploy, the backend deploy was re-implemented on **SSM Run Command** (bundle → S3 → `ssm send-command` → install/migrate/start/validate on the instance). A chain of first-deploy issues was worked through: the OIDC trust policy (repo name mismatch), an unset GitHub variable, a missing systemd unit, and a migration that ran before the venv existed. *(Backend deploy expected green by end of week.)*
- **Architecture diagram v6** — redrawn to address mentor feedback (explicit VPC, public/private subnets, NAT, Multi-AZ RDS, ALB + ASG across two AZs, GitHub outside the AWS Cloud boundary), with a numbered 22-step flow; the **Proposal §2.1 walkthrough was re-aligned** to match the diagram exactly.
- **Deployment runbook (Workshop §5)** authored, capturing the end-to-end build with every error and fix encountered.

---

### Team Meeting — 08/02/2026 *(planned)*

**Agenda:** backend deploy sign-off (SSM path), FE↔API wiring decision (Amplify `/api/*` proxy vs. `api.<domain>` with ACM), payment-Lambda track kickoff (Nguyen), booking hardening under concurrency (Thanh), manager/admin UI review (Danh & Hung).

---

### Glossary

| Abbreviation | Meaning |
| --- | --- |
| ACM | AWS Certificate Manager — issues/manages TLS certificates |
| ALB | Application Load Balancer |
| ASG | Auto Scaling Group |
| CDN | Content Delivery Network |
| NAT | Network Address Translation (Gateway) — outbound internet for private subnets |
| RDS | Amazon Relational Database Service |
| SG | Security Group — instance-level virtual firewall |
| SSM | AWS Systems Manager (Parameter Store, Run Command) |
| VPC | Virtual Private Cloud |
