---
title: "Workshop"
date: 2026-07-30
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Deploying the Court Booking Application on AWS

#### Purpose

This workshop demonstrates **how we implemented the Court Booking application on AWS** — from an empty account to a running system. It is the hands-on counterpart of the design in [2.1 Architecture](../2-proposal/2.1-architecture/): every resource created here maps to a component of that architecture.

The stack being deployed:

- **Backend** — FastAPI (Python 3.12) on EC2 behind an Application Load Balancer, deployed via GitHub Actions → CodeDeploy
- **Database** — Amazon RDS (PostgreSQL 16), schema managed by Alembic migrations
- **Authentication** — Amazon Cognito (user pool + groups as roles), wrapped by the backend
- **Frontend** — React + Vite SPA on AWS Amplify Hosting (built-in CI/CD)
- **Configuration** — SSM Parameter Store as the single source of runtime config/secrets

#### The two-account model

The project runs across **two AWS accounts** that meet only at the control plane (IAM, DNS, TLS) — there is no VPC peering, because there is exactly one workload VPC:

| Account | Holds |
| --- | --- |
| **Workload account** (Thanh) | VPC, EC2/ALB, RDS, Cognito, Amplify, S3, CodeDeploy |
| **DNS / guard account** (Hieu) | Route 53 hosted zone, ACM validation records, billing guardrails |

All CLI work targets the workload account through a cross-account **AssumeRole** (`--profile thanh`), set up in section 5.1. A single **`dev` environment** is used throughout (SSM path `/court-booking/dev/`, CodeDeploy group `dev`).

#### Deployment order

The sections follow the real dependency order — each one unblocks the next:

```
5.1 Identities (IAM · MFA · OIDC)     ← who is allowed to build & deploy
      → 5.2 CI pipeline (quality gate on every PR)
      → 5.3 Network + RDS             ← SG chain, database
      → 5.4 Config & secrets (SSM)    ← how the app receives its settings
      → 5.5 Cognito                   ← auth authority
      → 5.6 Backend deploy            ← EC2 + CodeDeploy walking skeleton
      → 5.7 Frontend (Amplify)        ← connect the SPA to the API
      → 5.8 Cleanup
```

#### Content

{{< children />}}
