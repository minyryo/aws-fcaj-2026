---
title: "Week 1 Worklog"
date: 2026-06-15
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---

### Week 1 Objectives:

- Connect and get acquainted with members of First Cloud AI Journey.
- Understand basic AWS services, how to use the Console & CLI.
- Study the core services that will be used in the court booking project: EC2, RDS, Cognito, Lambda, API Gateway, SNS, S3, and CloudWatch.
- Begin planning the hybrid architecture for the court booking application.

### Tasks to be carried out this week:

| Day | Task | Start Date | Completion Date | Reference Material |
| --- | ---- | ---------- | --------------- | ------------------- |
| 2 | - Get acquainted with FCAJ members <br> - Read and take note of internship unit rules and regulations <br> - Overview of the court booking project scope and deliverables <br> - **Team kick-off meeting**: present and align on hybrid architecture proposal; assign roles across FE, BE, and AWS Admin tracks | 06/15/2026 | 06/15/2026 | |
| 3 | - Learn about AWS and its types of services <br>&emsp; + Compute (EC2, Lambda) <br>&emsp; + Storage (S3, EBS) <br>&emsp; + Networking (VPC, ELB, API Gateway) <br>&emsp; + Database (RDS) <br>&emsp; + Security & Identity (Cognito, IAM) <br>&emsp; + Messaging (SNS) <br>&emsp; + Monitoring (CloudWatch) | 06/16/2026 | 06/16/2026 | <https://cloudjourney.awsstudygroup.com/> |
| 4 | - Create AWS Free Tier account <br> - Learn about AWS Console & AWS CLI <br> - **Practice:** <br>&emsp; + Create AWS account <br>&emsp; + Install & configure AWS CLI <br>&emsp; + How to use AWS CLI | 06/17/2026 | 06/17/2026 | <https://cloudjourney.awsstudygroup.com/> |
| 5 | - Learn basic EC2: <br>&emsp; + Instance types (focus: t3.small for FastAPI backend) <br>&emsp; + AMI <br>&emsp; + EBS <br>&emsp; + Auto Scaling Groups <br>&emsp; + Elastic Load Balancer <br> - SSH connection methods to EC2 <br> - Learn about Elastic IP <br> - Begin drafting hybrid architecture diagram for the court booking app | 06/18/2026 | 06/19/2026 | <https://cloudjourney.awsstudygroup.com/> |
| 6 | - **Practice:** <br>&emsp; + Launch an EC2 instance <br>&emsp; + Connect via SSH <br>&emsp; + Attach an EBS volume <br> - Review and finalize Week 1 architecture draft | 06/19/2026 | 06/19/2026 | <https://cloudjourney.awsstudygroup.com/> |

### Week 1 Achievements:

- Understood what AWS is and mastered the basic service groups relevant to the court booking project:
  - **Compute**: EC2 (for FastAPI backend), Lambda (for payment processing)
  - **Storage**: S3 (court photos and static assets), EBS (EC2 block storage)
  - **Networking**: VPC, Elastic Load Balancer, API Gateway
  - **Database**: RDS PostgreSQL (shared data layer for both the monolith and serverless paths)
  - **Security & Identity**: Cognito (user authentication), IAM (service permissions)
  - **Messaging**: SNS (booking confirmation notifications)
  - **Monitoring**: CloudWatch (unified logging for EC2 and Lambda)

- Successfully created and configured an AWS Free Tier account.

- Became familiar with the AWS Management Console and learned how to find, access, and use services via the web interface.

- Installed and configured AWS CLI on the computer, including:
  - Access Key
  - Secret Key
  - Default Region
  - Output format

- Used AWS CLI to perform basic operations such as:
  - Check account & configuration information
  - Retrieve the list of regions
  - View EC2 service and available instance types
  - Create and manage key pairs
  - Check information about running services

- Acquired the ability to connect between the web interface and CLI to manage AWS resources in parallel.

- Drafted the initial hybrid architecture design for the court booking application:
  - Identified the two traffic paths: EC2 monolith (core booking logic) and serverless (payment + notifications)
  - Decided on Amazon RDS (PostgreSQL) as the shared data layer to prevent double-booking race conditions
  - Identified CloudWatch as the unified monitoring solution across both deployment models

---

### Team Meeting — 06/15/2026

**Attendees:** All FCAJ team members

**Architecture Decision**

Hieu presented the hybrid AWS architecture proposal for the court booking application. The team aligned on the following breakdown of the three core features:

| Feature | Implementation |
| ------- | -------------- |
| User Authentication (Sign Up / Sign In) | Monolith (EC2) |
| Booking Management (Create / Edit / Cancel / View / Search) | Monolith (EC2) |
| Payment Processing | Serverless (Lambda + API Gateway) |

**Task Distribution**

| Track | Owner | Deliverable |
| ----- | ----- | ----------- |
| All members | Everyone | Review and provide feedback on the architecture proposal |
| Frontend | FE Team | Market research on similar apps; AI-assisted UX/UI design → design system (color palette, typography, icon set) and screen list per feature |
| Backend — Booking Management | Thanh | API documentation (endpoint name, input, output, use case) and DB design (tables, columns, data types) |
| Backend — User Authentication | Nguyen | API documentation and DB design for the authentication feature |
| Backend — Payment | Hieu | API documentation and DB design for the payment feature |
| AWS Administration | AWS Admin | Set up AWS Organization, accounts, IAM users, roles, and policies |

**Next Steps**
- FE team to complete market research and produce an initial design system by end of Week 2
- BE members to complete API docs and DB schema drafts by end of Week 2
- AWS Admin to have the base account structure ready before infrastructure work begins in Week 3
