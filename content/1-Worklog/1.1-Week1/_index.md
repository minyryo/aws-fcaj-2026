---
title: "Week 1 Worklog"
date: 2026-06-15
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---

### Week 1 Objectives:

- Connect and get acquainted with members of First Cloud AI Journey.
- Attend the team kick-off meeting and align on the hybrid AWS architecture proposal.
- Research monolith and serverless architecture patterns to understand the hybrid approach.
- Read an overview of AWS services referenced in the project architecture diagram.
- Create an AWS account.

### Tasks to be carried out this week:

| Day | Task | Start Date | Completion Date | Reference Material |
| --- | ---- | ---------- | --------------- | ------------------ |
| 2 | - **Team meeting**: attend kick-off meeting, present and align on hybrid architecture proposal; assign roles across FE, BE, and AWS Admin tracks | 06/15/2026 | 06/15/2026 | |
| 3 | - Research about monolith & serverless architecture | 06/16/2026 | 06/16/2026 | |
| 4 | - Read overview on mentioned AWS services on the architecture diagram: <br>&emsp; + EC2, Lambda (Compute) <br>&emsp; + RDS PostgreSQL (Database) <br>&emsp; + API Gateway, ELB (Networking) <br>&emsp; + Cognito, IAM (Security & Identity) <br>&emsp; + SNS (Messaging) <br>&emsp; + S3 (Storage) <br>&emsp; + CloudWatch (Monitoring) | 06/17/2026 | 06/18/2026 | <https://cloudjourney.awsstudygroup.com/> |
| 5 | - Create AWS Free Tier account | 06/19/2026 | 06/19/2026 | |

### Week 1 Achievements:

- Attended the team kick-off meeting; aligned on the hybrid AWS architecture splitting the court booking app into EC2 monolith (Auth + Booking) and serverless (Payment) paths.

- Understood the difference between monolith and serverless architecture patterns:
  - **Monolith**: single deployable unit, stateful, low-latency — suited for core booking logic on EC2
  - **Serverless**: event-driven, independently scalable, no server management — suited for payment webhook processing on Lambda

- Read an overview of all AWS services referenced in the architecture diagram:
  - **Compute**: EC2 (FastAPI backend), Lambda (payment processing)
  - **Database**: RDS PostgreSQL (shared data layer preventing double-booking race conditions)
  - **Networking**: API Gateway (webhook entry point), Elastic Load Balancer (EC2 traffic distribution)
  - **Security & Identity**: Cognito (user authentication), IAM (service permissions)
  - **Messaging**: SNS (booking confirmation notifications)
  - **Storage**: S3 (court photos and static assets)
  - **Monitoring**: CloudWatch (unified logging for EC2 and Lambda)

- Successfully created an AWS Free Tier account.

---

### Team Meeting — 06/20/2026

**Attendees:** Hieu, Danh, Thanh (Absent: Nguyen, Hung)

**Architecture Decision**

Hieu presented the hybrid AWS architecture proposal for the court booking application. The team aligned on the following breakdown of the three core features:

| Feature                                                     | Implementation                    |
| ----------------------------------------------------------- | --------------------------------- |
| User Authentication (Sign Up / Sign In)                     | Monolith (EC2)                    |
| Booking Management (Create / Edit / Cancel / View / Search) | Monolith (EC2)                    |
| Payment Processing                                          | Serverless (Lambda + API Gateway) |

**Task Distribution**

| Track                         | Owner     | Deliverable                                                                                                                                 |
| ----------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| All members                   | Everyone  | Review and provide feedback on the architecture proposal                                                                                    |
| Frontend                      | FE Team   | Market research on similar apps; AI-assisted UX/UI design → design system (color palette, typography, icon set) and screen list per feature |
| Backend — Booking Management  | Thanh     | API documentation (endpoint name, input, output, use case) and DB design (tables, columns, data types)                                      |
| Backend — User Authentication | Nguyen    | API documentation and DB design for the authentication feature                                                                              |
| Backend — Payment             | Hieu      | API documentation and DB design for the payment feature                                                                                     |
| AWS Administration            | AWS Admin | Set up AWS Organization, accounts, IAM users, roles, and policies                                                                           |

**Next Steps**

- FE team to complete market research and produce an initial design system by end of Week 2
- BE members to complete API docs and DB schema drafts by end of Week 2
- AWS Admin to have the base account structure ready before infrastructure work begins in Week 3
