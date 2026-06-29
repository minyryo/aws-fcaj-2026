---
title: "Proposal"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

In this section, you need to summarize the contents of the workshop that you **plan** to conduct.

# Court Booking Application

## A Hybrid AWS Architecture for Scalable and Reliable Court Reservation

### 1. Executive Summary

This project designs and deploys a hybrid AWS architecture for a court booking application that allows players to browse, search, and reserve sports courts online. Core application logic runs on a monolithic EC2 deployment, while payment processing and notifications are handled through a serverless event-driven path. Both paths share a single Amazon RDS (PostgreSQL) instance as the source of truth, preventing data consistency issues such as double-bookings. User identity is managed by Amazon Cognito, and system health is monitored centrally via Amazon CloudWatch.

### 2. Problem Statement

#### What's the Problem?

Managing court reservations manually is error-prone and difficult to scale. Without a centralized system, double-bookings are common, payment confirmation is slow and unreliable, and players have no real-time visibility into court availability.

#### The Solution

A hybrid architecture combines the reliability of EC2-hosted application logic with the elasticity of serverless payment processing. Player-facing requests (browsing, searching, booking) are handled by a FastAPI backend on EC2, behind an Elastic Load Balancer with Auto Scaling. Payment events are routed through Amazon API Gateway to AWS Lambda, which confirms payments and writes bookings back to RDS, then notifies players via Amazon SNS. Court photos and static assets are served from Amazon S3.

#### Benefits and Return on Investment

- Prevents double-bookings through relational integrity enforced at the RDS layer
- Scales automatically during peak hours without over-provisioning EC2 capacity
- Reduces payment infrastructure cost by using Lambda's pay-per-invocation model for bursty, low-frequency payment events
- Gives the team a single monitoring view across both deployment models via CloudWatch
- Demonstrates competency in EC2, Lambda, and API Gateway - directly aligned with the AWS FCJ program Week 3 deliverable

### 3. Core Application Features

The application is scoped to three main features, each assigned to a deployment model based on its workload characteristics:

| Feature                                                     | Deployment Model                  | Rationale                                                              |
| ----------------------------------------------------------- | --------------------------------- | ---------------------------------------------------------------------- |
| User Authentication (Sign Up / Sign In)                     | Monolith (EC2)                    | Frequent, latency-sensitive; benefits from persistent compute          |
| Booking Management (Create / Edit / Cancel / View / Search) | Monolith (EC2)                    | Core business logic requiring relational integrity and complex queries |
| Payment Processing                                          | Serverless (Lambda + API Gateway) | Bursty, event-driven, and isolated — ideal for pay-per-invocation      |

### 4. Solution Architecture

The system follows a hybrid pattern with two distinct traffic paths sharing one data layer:

**Core backend (EC2 monolith)**  
Player requests are routed through an Elastic Load Balancer to an Auto Scaling EC2 instance group running the FastAPI application. Amazon Cognito handles authentication, verifying tokens before requests reach the backend. The application reads and writes booking, user, and court data directly to Amazon RDS (PostgreSQL), and stores court photos and static assets in Amazon S3.

**Serverless path (event-driven)**  
Payment is handled through Amazon API Gateway, which invokes an AWS Lambda function to process the transaction. Once payment is confirmed, a second Lambda function writes the booking confirmation to RDS and publishes an event to Amazon SNS, which notifies the player. This path suits Lambda because payment events are bursty, isolated, and do not require persistent compute.

**Shared data layer**  
Both the monolith and the serverless functions write to the same Amazon RDS (PostgreSQL) database. A relational database was chosen over DynamoDB because bookings require transactional guarantees - specifically, preventing two players from reserving the same court slot simultaneously.

**Monitoring**  
Amazon CloudWatch collects logs and metrics from both EC2 instances and Lambda functions, providing a unified view for troubleshooting and performance tracking.

![Court Booking Hybrid Architecture - Overview](/images/2-Proposal/court_booking_hybrid_v2.png)

![Court Booking Hybrid Architecture - Detailed Flow](/images/2-Proposal/court_booking_hybrid_v1.png)

### AWS Services Used

- **Amazon EC2 + Auto Scaling**: Hosts the FastAPI backend; scales horizontally under load.
- **Elastic Load Balancer**: Distributes incoming player traffic across EC2 instances.
- **Amazon Cognito**: Manages user identity and token-based authentication.
- **Amazon RDS (PostgreSQL)**: Single source of truth for bookings, users, and courts.
- **Amazon S3**: Stores court photos and static assets.
- **Amazon API Gateway**: Exposes the payment endpoint to Lambda.
- **AWS Lambda**: Processes payments and writes booking confirmations (two functions).
- **Amazon SNS**: Delivers booking confirmation notifications to players.
- **Amazon CloudWatch**: Centralized logging and metrics for EC2 and Lambda.

### Component Design

- **Load Balancing**: ELB routes HTTP traffic to the EC2 Auto Scaling group.
- **Application Layer**: FastAPI on EC2 handles browsing, searching, and booking logic.
- **Authentication**: Cognito verifies JWTs before requests reach the backend.
- **Data Layer**: RDS enforces relational integrity and prevents concurrent booking conflicts.
- **Asset Storage**: S3 serves court images and other static content.
- **Payment Flow**: API Gateway → Lambda (payment) → Lambda (confirmation write to RDS) → SNS notification.
- **Observability**: CloudWatch aggregates logs and metrics from both EC2 and Lambda.

### 5. Design Rationale: Why Hybrid?

A fully monolithic design would force every workload - including bursty, isolated payment events - through the same EC2 fleet, regardless of whether persistent compute is needed. A fully serverless design would have added significant friction during early development (simulating Lambda cold starts and API Gateway routing locally is non-trivial), which was a constraint given the eight-week project timeline.

The hybrid approach lets each part of the system play to its strength. Core booking and search logic stays on EC2 because it is the most frequently accessed, latency-sensitive part of the application - a persistent server avoids cold start delays and is easier to develop and debug locally. Payment processing and notifications, by contrast, are naturally event-driven and infrequent relative to browsing traffic, making Lambda and SNS a natural fit: the team only pays for compute when a payment event actually occurs.

This also allowed the team to demonstrate competency in both deployment models within the same project - directly reflecting the AWS FCJ program's Week 3 deliverable of deploying via EC2, Lambda, and API Gateway - without overcomplicating the core application architecture.

### 6. Timeline & Milestones

| Week | Deliverable                                                   |
| ---- | ------------------------------------------------------------- |
| 1–2  | Set up VPC, EC2, RDS, Cognito; deploy FastAPI skeleton        |
| 3    | Implement EC2 Auto Scaling and ELB; integrate Cognito auth    |
| 4    | Build booking logic; enforce RDS transactional constraints    |
| 5    | Implement API Gateway + Lambda payment flow                   |
| 6    | Wire SNS notifications; connect confirmation Lambda to RDS    |
| 7    | Set up CloudWatch dashboards and alarms; S3 asset integration |
| 8    | End-to-end testing, performance tuning, final documentation   |

### 7. Budget Estimation

| Service                                | Estimated Monthly Cost |
| -------------------------------------- | ---------------------- |
| Amazon EC2 (t3.small × 2)              | ~$15.00                |
| Amazon RDS (db.t3.micro, Single-AZ)    | ~$13.00                |
| Elastic Load Balancer                  | ~$16.00                |
| Amazon Cognito (≤50,000 MAU free tier) | $0.00                  |
| AWS Lambda (pay-per-invocation)        | ~$0.00–$1.00           |
| Amazon API Gateway                     | ~$0.01                 |
| Amazon SNS                             | ~$0.00                 |
| Amazon S3                              | ~$0.50                 |
| Amazon CloudWatch                      | ~$1.00                 |
| **Total**                              | **~$45–$47/month**     |

Costs can be reduced significantly by using EC2 Reserved Instances or Savings Plans for the production environment.

### 8. Risk Assessment

#### Risk Matrix

- **Double-booking race condition**: High impact, low probability - mitigated by RDS transactions and row-level locking.
- **EC2 instance failure**: High impact, low probability - mitigated by Auto Scaling and ELB health checks.
- **Lambda cold start latency on payment**: Medium impact, low probability - mitigated by provisioned concurrency if needed.
- **Cost overrun**: Medium impact, low probability - mitigated by CloudWatch billing alarms and Auto Scaling cooldown policies.

#### Mitigation Strategies

- Use `SELECT FOR UPDATE` or PostgreSQL advisory locks to prevent concurrent booking conflicts at the database level.
- Configure ELB health checks to automatically replace unhealthy EC2 instances.
- Set AWS budget alerts to notify when monthly spend exceeds a defined threshold.

### 9. Expected Outcomes

- **Technical**: A fully functional court booking system with real-time availability, secure payment, and instant notification - deployed on a production-grade hybrid AWS architecture.
- **Educational**: Demonstrated hands-on experience with EC2, RDS, Cognito, Lambda, API Gateway, SNS, S3, and CloudWatch within a single cohesive project.
- **Operational**: Auto Scaling ensures the system handles traffic spikes without manual intervention; CloudWatch provides ongoing visibility into system health.
