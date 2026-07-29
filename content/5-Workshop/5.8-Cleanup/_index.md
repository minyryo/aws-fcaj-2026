---
title: "Cleanup"
date: 2026-07-30
weight: 8
chapter: false
pre: " <b> 5.8. </b> "
---

When the project winds down, resources are removed in **reverse dependency order** — the opposite of how they were built. Only the always-free, global pieces stay.

#### Teardown order

| # | Resource | How | Why this order |
| - | --- | --- | --- |
| 1 | CodeDeploy app + deployment group | console / CLI | stops deployments targeting instances |
| 2 | Auto Scaling Group (desired = 0) → launch template | EC2 console | terminates instances cleanly before their SGs go |
| 3 | ALB + target group | EC2 console | hourly billing stops the moment it's deleted |
| 4 | RDS instance | `delete-db-instance --final-db-snapshot-identifier court-booking-final` | **take the final snapshot** — it preserves the data at ~$0 while stopped |
| 5 | Cognito user pool | console | user data is in RDS; the pool is recreatable |
| 6 | Amplify app | console | stops FE builds/hosting |
| 7 | S3 buckets (deploy artifacts, assets) | empty, then delete | buckets must be empty first |
| 8 | SSM parameters | `delete-parameters` under `/court-booking/dev/` | removes the stored DB URL (contains the password) |
| 9 | Security groups + DB subnet group | VPC console | deletable only after instances/RDS are gone (dependency errors otherwise) |

#### What stays (deliberately)

- **IAM roles, OIDC provider, policies** — global, free, and the record of the identity design
- **Route 53 hosted zone** (other account) — $0.50/mo; keep while the domain is owned
- **The final RDS snapshot** — the only copy of the data; delete last, consciously, or share it to the other account first (cross-account snapshot copy) as the off-account backup

{{% notice warning %}}
The two **always-on billers** in this stack are the **ALB** (hourly, ~$16–18/mo beyond free tier) and **RDS** (750 free hours/mo covers one instance — a second one bills). If pausing the project rather than ending it: delete the ALB, stop the RDS instance (auto-restarts after 7 days), and set the ASG to 0 — that reduces the bill to storage cents while keeping everything recreatable.
{{% /notice %}}

{{% notice tip %}}
Verify the account is actually quiet afterwards: **Billing → Bills** should show $0 forecast, and a **Budgets alert** (created during setup) acts as the tripwire if something was missed.
{{% /notice %}}
