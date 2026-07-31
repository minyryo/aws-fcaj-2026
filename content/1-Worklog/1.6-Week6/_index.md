---
title: "Week 6 Worklog"
date: 2026-07-20
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Week 6 Objectives:

- Establish the **cross-account AWS identity foundation** (IAM user + MFA, cross-account AssumeRole, GitHub OIDC provider, keyless deploy roles).
- Stand up the **CI quality gate** (`ci.yml`) in the backend repo so every PR is tested against a real database.
- **Unblock the deployment track** by implementing the remaining backend API surface — endpoints distributed to teammates by domain, brought forward so the CI/CD and deploy work can proceed against a complete app.

### Tasks to be carried out this week:

| Day | Task | Start Date | Completion Date | Reference Material |
| --- | ---- | ---------- | --------------- | ------------------ |
| 2 | Local CLI identity: `hieu-cli` IAM user + MFA; **cross-account `court-booking-admin` role** (AssumeRole into the workload account); verified with `get-caller-identity` | 07/20/2026 | 07/20/2026 | [CI/CD guide §1.0](/2-proposal/2.2-deployment/) |
| 3 | **GitHub OIDC** identity provider + least-privilege `gh-deploy-backend` / `gh-deploy-lambdas` roles (keyless CI — no AWS keys in GitHub) | 07/21/2026 | 07/21/2026 | [CI/CD guide §1.1–1.3](/2-proposal/2.2-deployment/) |
| 4–5 | **`ci.yml` pipeline**: path-filtered jobs, Postgres 16 service container, `alembic upgrade head`, ruff + pytest; fixed a ruff version-drift failure (pinned `ruff==0.15.21`) and a test state-pollution bug (session-scoped auto-reseed in `conftest.py`) | 07/22/2026 | 07/23/2026 | [CI/CD guide §5](/2-proposal/2.2-deployment/) |
| 6 | Implemented the **remaining backend routers** — auth, user profile, court management, manager dashboard, admin operations, payments — plus read endpoints (court detail, list-my-courts, schedule/images/blackouts, admin user list); 52 tests green, `ruff` clean | 07/24/2026 | 07/24/2026 | [Proposal §6.1, §6.5, §6.6](/2-proposal/2.1-architecture/) |
| 7 | **Event participation** — FCAJ x Agentic AI Build Week (Bitexco Tower) | 07/25/2026 | 07/25/2026 | [§4.3](/4-eventparticipated/4.3-event3/) |
| CN | **Team meeting (07/26/2026)** — CI/CD + API progress review; ownership hand-back | 07/26/2026 | 07/26/2026 | |

### Week 6 Achievements:

- **Cross-account identity foundation complete** — a human setup identity (MFA-gated AssumeRole) plus two keyless CI roles via GitHub OIDC, so deployments never carry static AWS keys → [CI/CD guide §1](/2-proposal/2.2-deployment/).
- **CI quality gate live** — every PR now runs ruff + pytest against a real Postgres 16 container with migrations applied. Two failures were diagnosed and fixed: an unpinned-`ruff` version drift (local passed, CI failed on 0.16.0's expanded default rules → pinned the version), and flaky booking tests caused by seed-row state pollution (fixed with an idempotent per-session reseed fixture).
- **Backend API surface complete** — all remaining routers implemented and tested (52 passing). These endpoints are **owned by teammates by domain** (Admin Operations + Cognito role → Nguyen, Booking + Revenue → Thanh); I brought them forward so the CI/CD and deployment work I own is not blocked waiting on a complete app. Ownership returns to each teammate for review, refinement, and the payment-Lambda split.
- **`openapi.json` regenerated** as the single FE↔BE contract; the frontend (Danh & Hung) consumes it for type generation.

---

### Team Meeting — 07/26/2026

**Attendees:** Hieu, Thanh, Nguyen, Danh, Hung
**Absent:** None

**Presentations**

- **Hieu** demoed the CI pipeline (Postgres-backed ruff + pytest on every PR), walked through the cross-account identity setup (MFA AssumeRole + OIDC deploy roles), and showed the completed backend API surface with the regenerated `openapi.json` contract.
- **Thanh** and **Nguyen** reviewed their domain endpoints as implemented, confirming the contract matches the §6.5 / §6.6 design.
- **Danh & Hung** showed the frontend consuming the real endpoints behind the mock/real switch, with type generation from `openapi.json`.

**Decisions & workload distribution**

| # | Action | Owner | Notes |
| - | ------ | ----- | ----- |
| 1 | Own the **AWS infrastructure + deployment** track next week (network, RDS, Cognito, SSM, EC2 + CI/CD deploy) | Hieu | Priority — the app is code-complete; the gap is now infra |
| 2 | Review + refine the **Admin Operations** router and the **Cognito-first role** endpoint against §6.6; own the **payment-Lambda** split when the serverless track begins | Nguyen | His auth/admin domain |
| 3 | Review + harden the **booking + revenue** endpoints; validate the double-booking exclusion constraint under concurrency | Thanh | His booking domain |
| 4 | Build the **manager/admin/profile UI** screens against the live endpoints; prepare Amplify hosting | Danh & Hung | FE domain |

---

### Glossary

| Abbreviation | Meaning |
| --- | --- |
| API | Application Programming Interface |
| AssumeRole | AWS STS action to obtain temporary credentials for another role/account |
| CI/CD | Continuous Integration / Continuous Delivery |
| IAM | AWS Identity and Access Management |
| MFA | Multi-Factor Authentication |
| OIDC | OpenID Connect — used for keyless GitHub → AWS authentication |
| PR | Pull Request |
