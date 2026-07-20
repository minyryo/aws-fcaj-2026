---
title: "Week 5 Worklog"
date: 2026-07-13
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Week 5 Objectives:

- Finalize the CI/CD & deployment design (two-repo layout, two-account strategy) — details in [Proposal §2.2](/2-proposal/2.2-deployment/).
- Revise the unified design for the new **Court Manager** role — details in [Proposal §2.1, §6.5](/2-proposal/2.1-architecture/).
- Initialize both code repositories and scaffold the backend (FastAPI + Alembic); implement the 7-table schema locally.
- Begin local frontend–backend integration.

### Tasks to be carried out this week:

| Day | Task                                                                                                                                                                                                                 | Start Date | Completion Date | Reference Material                                                       |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | --------------- | ------------------------------------------------------------------------ |
| 2   | - Decide the **two-account deployment strategy** (dev on personal AWS account, prod on sponsored credits) and the phasing plan (repos & CI first, IaC/CD later)                                                       | 07/13/2026 | 07/13/2026      | [Proposal §2.2](/2-proposal/2.2-deployment/)                             |
| 3   | - **Court Manager design revision** (§6.5): role rename, 10 new endpoints, `court_schedules`/`court_blackouts` tables, updated ERD <br> - Create `court-booking-backend` & `court-booking-frontend` repos; scaffold BE per the step-by-step guide (§2.4); set up pyenv-virtualenv | 07/14/2026 | 07/14/2026      | [Proposal §2.1](/2-proposal/2.1-architecture/)                           |
| 6   | - Implement SQLAlchemy models for the 7-table schema; iterative review against §6.2 (typing, constraints, nullability, FK indexes)                                                                                    | 07/17/2026 | 07/17/2026      | [Proposal §6.2](/2-proposal/2.1-architecture/)                           |
| 7   | - Local Postgres via Docker Compose; wire Alembic and apply the first migrations (`btree_gist`, core tables) <br> - Add CORS; verify the FE service layer (axios client, mock/real switcher) against the §6.1 contract | 07/18/2026 | 07/18/2026      | [CI/CD guide Part 0.5](/2-proposal/2.2-deployment/)                      |
| CN  | - **Team meeting (07/19/2026)**: BE progress demo; review of Nguyen's proposed API additions                                                                                                                          | 07/19/2026 | 07/19/2026      |                                                                          |

### Week 5 Achievements:

Full design detail lives in the Proposal — this list only records what was done, with links.

- **Deployment design finalized** — two repos along team ownership, GitHub Actions orchestrating Amplify / CodeDeploy / SAM, the OpenAPI contract check, and the two-account (dev/prod) strategy → [Proposal §2.2](/2-proposal/2.2-deployment/).
- **Court Manager design revision** — role rename, API 15 → 25 endpoints, schema 5 → 7 tables, updated Mermaid + drawio ERD → [Proposal §6.5](/2-proposal/2.1-architecture/).
- **Both repositories live**; backend scaffolded on branch `chore/initial-setup`: FastAPI skeleton with `/api/v1/health`, 7-table SQLAlchemy models faithful to §6.2 (verified by live mapper configuration), Alembic chain applied and verified (`btree_gist` → core tables incl. the double-booking exclusion constraint).
- **FE service layer verified** against the §6.1 contract: shared axios client with 401-refresh flow, real/mock service twins behind `VITE_USE_MOCK_API` — connection plan documented as CI/CD guide Part 0.5.
- **Team documentation shipped**: backend README (setup, conventions, troubleshooting table, design change log), `alembic-working-guide.md`, and the expanded `cicd-setup-guide.md` (scaffold + FE–BE connection parts).

---

### Team Meeting — 07/19/2026

**Attendees:** Hieu, Thanh, Nguyen, Danh, Hung
**Absent:** None

**Presentations**

- **Hieu** demoed the backend scaffold: repo structure, 7-table schema migrations applied to local Postgres, and the deployment guide walkthrough (two-repo CI/CD, two-account strategy).
- **Nguyen** proposed 6 additional endpoints extending the unified API design (working doc `ADJ_APIs.md`): **Admin Operations** (court approval queue, approve/reject, user role management), **Manager Analytics** (revenue), and **User Profile** (view/update — deferred as low priority).

**Review summary of Nguyen's API additions**

1. **The admin approval endpoints fill a real gap**: §6.5 introduced the `PENDING` → `ACTIVE`/`REJECTED` court lifecycle but defined no admin API to drive it. `GET /admin/courts` (queue) + a review endpoint complete the flow, with an SNS notification to the court manager on decision.
2. **The revenue endpoint backs the manager dashboard** promised in §6.5. Agreed definition: aggregate from **`payments` with `status = 'SUCCESS'`** (refunds fall out automatically) joined to the caller's courts — not from booking totals — and scoped by `courts.owner_id` per the IDOR rule, with optional `group_by=day|court` for charts.
3. **Role changes must go through Cognito first**: `users.role` is only a cache — the endpoint must call Cognito `AdminAddUserToGroup`/`AdminRemoveUserFromGroup`, then update the DB row, or JWT claims and DB drift apart.
4. **Naming standardization**: `GET/PUT /users/me` (matching the existing `/bookings/me` pattern) instead of verb-style `/users/update-profile`.
5. **Schema addition needed**: `courts.rejection_reason` (nullable) so a rejected court's reason survives beyond the notification.

**Decisions & next steps**

| # | Action | Owner |
| - | ------ | ----- |
| 1 | Adopt all 6 endpoints with the adjustments above; integrate into the Proposal as revision §6.6 | Hieu & Nguyen |
| 2 | Continue booking routers on the scaffolded schema | Thanh |
| 3 | Complete the FE–BE connectivity proof (health check through CORS) and continue per-feature connection | Danh & Hung |
| 4 | User-profile endpoints deferred (low priority, revisit after core features) | — |

---

### Glossary

| Abbreviation | Meaning |
| --- | --- |
| API | Application Programming Interface — the contract through which software components communicate |
| AWS | Amazon Web Services — Amazon's cloud computing platform |
| BE | Backend — the server-side part of the application |
| CI/CD | Continuous Integration / Continuous Delivery — automated building, testing, and deployment |
| CORS | Cross-Origin Resource Sharing — browser mechanism allowing a page to call APIs on another origin |
| DB | Database |
| ERD | Entity-Relationship Diagram — visual model of database tables and their relationships |
| FE | Frontend — the client-side part of the application |
| FK | Foreign Key — a column referencing another table's primary key |
| IaC | Infrastructure as Code — declaring infrastructure in versioned files |
| IDOR | Insecure Direct Object Reference — accessing another user's resources by manipulating object IDs |
| JWT | JSON Web Token — signed token carrying identity claims |
| SNS | Amazon Simple Notification Service — pub/sub messaging and notifications |
