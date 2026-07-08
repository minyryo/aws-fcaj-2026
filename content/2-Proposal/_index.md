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

![Court Booking Hybrid Architecture (v3)](/images/2-Proposal/court_booking_hybrid_v3.png)

[View the full-resolution SVG version](/images/2-Proposal/court_booking_hybrid_v3.svg)

#### Architecture Walkthrough — 14 Steps

**Frontend & authentication (steps 1–3)**

| Step | Flow                          | Explanation                                                                                                                                              |
| ---- | ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Player → AWS Amplify          | **Visit app** — the player opens the application; the React + Vite frontend is served globally from AWS Amplify's CDN.                                    |
| 2    | Player → Amazon Cognito       | **Login** — the player signs in (email/password or Google/Facebook) against the Cognito User Pool and receives a short-lived JWT plus a refresh token.    |
| 3    | Amplify frontend → ELB        | **API calls** — the authenticated frontend calls the backend REST APIs (auth, courts, bookings, payments) with the JWT attached as a Bearer token.        |

**Core backend — EC2 monolith (steps 4–6)**

| Step | Flow                | Explanation                                                                                                                                                     |
| ---- | ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 4    | ELB → Amazon EC2    | **Forward request** — the Elastic Load Balancer distributes traffic across the FastAPI instances in the Auto Scaling Group, which scales the fleet under load.     |
| 5    | EC2 → Amazon RDS    | **Read / write booking data** — FastAPI reads and writes `users`, `courts`, `bookings`, and `payments` in PostgreSQL, applying the row-level lock + exclusion constraint to prevent double-booking. |
| 6    | EC2 → Amazon S3     | **Read / write photos** — court photos and static assets are stored in and served from S3.                                                                         |

**Serverless payment path (steps 7–12)**

| Step | Flow                                  | Explanation                                                                                                                                     |
| ---- | ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| 7    | Player → Amazon API Gateway           | **Pay** — the player completes checkout with the external payment gateway, whose webhook callback lands on the API Gateway payment endpoint.        |
| 8    | API Gateway → Lambda (Process payment) | **Invoke** — API Gateway invokes the payment-processing Lambda with the webhook payload.                                                            |
| 9    | Lambda → Amazon RDS                   | **Write payment status** — the Lambda validates the webhook and updates the `payments` record (`SUCCESS`/`FAILED`, `transaction_id`, `gateway_response`). |
| 10   | Lambda → Lambda (Confirm booking)     | **Trigger** — on successful payment, the processing function triggers the booking-confirmation Lambda.                                              |
| 11   | Lambda → Amazon RDS                   | **Confirm slot** — the confirmation Lambda sets `bookings.status = 'CONFIRMED'` (or `CANCELLED` on failure), finalizing the reserved slot.          |
| 12   | Lambda → Amazon SNS                   | **Publish** — the confirmation Lambda publishes the booking result to the SNS notifications topic.                                                  |

**Notifications (steps 13–14)**

| Step | Flow                    | Explanation                                                                                                       |
| ---- | ----------------------- | -------------------------------------------------------------------------------------------------------------------- |
| 13   | SNS → Amazon SES        | **Send email** — SNS fans out to SES, which sends the booking/payment confirmation email to the player.               |
| 14   | SNS → Player            | **Push notify** — SNS simultaneously delivers a push notification back to the player's client for instant feedback.   |

**Supporting flows (unnumbered):** the developer pushes code to the GitHub source repo (*Push code*), which Amplify picks up for **CI/CD deploy**; Amplify performs an **Auth check** against Cognito before serving protected routes; the Auto Scaling Group **scales** the EC2 fleet; EC2 and both Lambdas **push logs** to Amazon CloudWatch, where the developer **views logs and metrics**.

### AWS Services Used

- **AWS Amplify**: Hosts the React + Vite frontend on a global CDN, with CI/CD deployment from the GitHub source repo.
- **Amazon EC2 + Auto Scaling**: Hosts the FastAPI backend; scales horizontally under load.
- **Elastic Load Balancer**: Distributes incoming player traffic across EC2 instances.
- **Amazon Cognito**: Manages user identity and token-based authentication.
- **Amazon RDS (PostgreSQL)**: Single source of truth for bookings, users, and courts.
- **Amazon S3**: Stores court photos and static assets.
- **Amazon API Gateway**: Exposes the payment endpoint to Lambda.
- **AWS Lambda**: Processes payments and writes booking confirmations (two functions).
- **Amazon SNS**: Delivers booking confirmation notifications to players.
- **Amazon SES**: Sends booking and payment confirmation emails.
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

### 6. Unified API & Database Design

This section consolidates the three individual backend deliverables — Hieu's payment design (Week 2), Thanh's booking design, and Nguyen's authentication design (both Week 3) — into a single canonical specification, as assigned in the 07/05/2026 team meeting.

**Guiding principle: maximise the use of AWS managed services.** Wherever an AWS service natively covers a concern, the design delegates to it instead of rebuilding it in the database or application code. The largest consequence is in identity: Amazon Cognito is the sole credential and session authority, which lets the relational schema stay at **5 tables**.

| Concern                                | Handled by                                                            | Replaces (from individual designs)                     |
| -------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------- |
| Credential storage (passwords)         | Cognito User Pool                                                     | `users.password_hash` column                            |
| Social login (Google / Facebook)       | Cognito federated identity providers (Hosted UI)                      | `user_identities` table                                 |
| Session lifecycle, refresh, revocation | Cognito refresh tokens + `GlobalSignOut`                              | `refresh_tokens` table                                  |
| Roles & permissions (RBAC)             | Cognito Groups (`cognito:groups` claim in the JWT)                    | `roles` + `user_roles` bridge tables                    |
| Multi-role users (player + court_owner)| A user may belong to multiple Cognito Groups                          | Many-to-many `user_roles` design                        |

#### 6.1 Unified API Documentation (15 endpoints)

**Authentication — 5 endpoints** (contracts from Nguyen's design, re-implemented as FastAPI wrappers around Cognito)

| # | Endpoint                | Method | Input                                        | Output                                              | Use Case                                                                                                                                       |
| - | ----------------------- | ------ | -------------------------------------------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | `/api/v1/auth/register` | `POST` | `email`, `password`, `full_name`             | `user_id`, `message: "Success"`                      | Registers a new account via Cognito `SignUp`; on confirmation, inserts a profile row into `users` linked by `cognito_sub`.                       |
| 2 | `/api/v1/auth/login`    | `POST` | `email`, `password`                          | `access_token` (JWT), `refresh_token`, `user_data`   | Standard login via Cognito `InitiateAuth`. Cognito issues a short-lived JWT and a revokable refresh token — no session table needed.             |
| 3 | `/api/v1/auth/oauth`    | `POST` | `provider` (google/facebook), `auth_code`    | `access_token` (JWT), `refresh_token`, `user_data`   | OAuth via Cognito federated identity providers. First social login auto-upserts the `users` profile row.                                        |
| 4 | `/api/v1/auth/refresh`  | `POST` | `refresh_token`                              | `access_token` (new JWT)                             | Exchanges the Cognito refresh token for a new access token (`REFRESH_TOKEN_AUTH` flow) when the ~15-minute JWT expires.                          |
| 5 | `/api/v1/auth/logout`   | `POST` | Bearer token (header)                        | `204 No Content`                                     | Calls Cognito `GlobalSignOut`, revoking all refresh tokens for the user — instant remote logout without a DB write.                              |

**Booking — 6 endpoints** (from Thanh's design, input/output specifications completed)

| # | Endpoint                                  | Method   | Input                                                            | Output                                                    | Use Case                                                                                                                     |
| - | ----------------------------------------- | -------- | ----------------------------------------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| 1 | `/api/v1/courts`                          | `GET`    | `sport_type`, `address`, `page`, `limit` (query)                  | `{ data: [...courts], total, page, limit }`                | Players search and browse active sports courts by location or sport type.                                                     |
| 2 | `/api/v1/courts/{court_id}/availability`  | `GET`    | `court_id` (path), `date` (query)                                 | `{ court_id, date, booked_slots: [{start_time, end_time}] }` | Retrieve booked time slots of a court for a selected date so the frontend can disable those slot buttons.                     |
| 3 | `/api/v1/bookings`                        | `POST`   | Bearer token; `court_id`, `start_time`, `end_time`, `note` (opt.) | `booking_id`, `total_amount`, `status: "PENDING"` — or `409 Conflict` | Creates a temporary booking (PENDING) during checkout. Applies row-level locking to prevent double-booking.          |
| 4 | `/api/v1/bookings/{booking_id}`           | `PUT`    | Bearer token; `start_time`, `end_time`                            | Updated booking object                                     | Player changes the slot of an unpaid booking (PENDING only). Re-runs the overlap check.                                        |
| 5 | `/api/v1/bookings/{booking_id}`           | `DELETE` | Bearer token; `booking_id` (path)                                 | `204 No Content` (`status → CANCELLED`)                    | Cancels a booking (unpaid pending, or confirmed per the court cancellation policy).                                            |
| 6 | `/api/v1/bookings/me`                     | `GET`    | Bearer token; `status`, `page`, `limit` (query)                   | `{ data: [...bookings], total, page, limit }`              | Retrieves the authenticated user's current bookings and history (`user_id` extracted from the JWT).                            |

**Payment — 4 endpoints** (from Hieu's Week 2 design, unchanged)

| # | Endpoint                         | Method | Input                                                             | Output                                                | Use Case                                                                                                                     |
| - | -------------------------------- | ------ | ------------------------------------------------------------------ | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| 1 | `/api/v1/payments`               | `POST` | `booking_id`, `amount`, `payment_method`                           | `payment_id`, `checkout_url`, `status: "PENDING"`      | Player initiates payment after confirming a booking; creates a PENDING payment record and returns a checkout URL.             |
| 2 | `/api/v1/payments/webhook`       | `POST` | `payment_id`, `transaction_id`, `status`, `gateway_data`           | `{ success: boolean }`                                 | Gateway callback — entry point of the serverless path: API Gateway → Lambda updates RDS → SNS notifies the player.            |
| 3 | `/api/v1/payments/{payment_id}`  | `GET`  | `payment_id` (path), Bearer token                                  | Full payment object                                    | Player or admin checks the status of a specific payment.                                                                       |
| 4 | `/api/v1/payments`               | `GET`  | Bearer token; `page`, `limit`                                      | `{ data: [...payments], total, page, limit }`          | Player views their paginated payment history.                                                                                  |

#### 6.2 Unified Database Design (5 tables)

The only structurally revised table is `users` — reshaped around Cognito as the identity authority:

**`users`** (revised)

| Column        | Data Type      | Constraints                  | Description                                                                                  |
| ------------- | -------------- | ---------------------------- | --------------------------------------------------------------------------------------------- |
| `id`          | `UUID`         | `PRIMARY KEY`                | Unique user identifier                                                                        |
| `cognito_sub` | `VARCHAR(255)` | `UNIQUE, NOT NULL`           | Cognito User Pool subject ID — the sole link between JWTs and the local profile row           |
| `email`       | `VARCHAR(255)` | `UNIQUE, NOT NULL`           | Login email (synced from Cognito)                                                             |
| `full_name`   | `VARCHAR(255)` | `NOT NULL`                   | Display name                                                                                  |
| `phone`       | `VARCHAR(20)`  |                              | Contact number                                                                                |
| `avatar_url`  | `VARCHAR(500)` |                              | S3 URL of profile photo                                                                       |
| `role`        | `VARCHAR(20)`  | `DEFAULT 'player'`           | Primary role cached for DB queries; authorization uses the `cognito:groups` JWT claim         |
| `is_active`   | `BOOLEAN`      | `DEFAULT true`               | Soft delete / suspension flag                                                                 |
| `created_at`  | `TIMESTAMP`    | `DEFAULT NOW()`              | Account creation time                                                                         |
| `updated_at`  | `TIMESTAMP`    | `DEFAULT NOW()`              | Last profile update                                                                           |

> `password_hash` is intentionally absent: credentials live exclusively in Cognito. `courts`, `court_images`, `bookings`, and `payments` are carried over unchanged — full column specifications are in the [Week 2 worklog](/1-worklog/1.2-week2/).

{{<mermaid>}}
erDiagram
users {
UUID id PK
VARCHAR cognito_sub UK
VARCHAR email UK
VARCHAR full_name
VARCHAR phone
VARCHAR avatar_url
VARCHAR role
BOOLEAN is_active
TIMESTAMP created_at
TIMESTAMP updated_at
}
courts {
UUID id PK
UUID owner_id FK
VARCHAR name
TEXT description
VARCHAR address
VARCHAR sport_type
DECIMAL price_per_hour
VARCHAR status
TIMESTAMP created_at
TIMESTAMP updated_at
}
court_images {
UUID id PK
UUID court_id FK
VARCHAR image_url
BOOLEAN is_primary
TIMESTAMP created_at
}
bookings {
UUID id PK
UUID user_id FK
UUID court_id FK
TIMESTAMP start_time
TIMESTAMP end_time
DECIMAL total_amount
VARCHAR status
TEXT note
TIMESTAMP created_at
TIMESTAMP updated_at
}
payments {
UUID id PK
UUID booking_id FK
UUID user_id FK
DECIMAL amount
VARCHAR currency
VARCHAR status
VARCHAR payment_method
VARCHAR transaction_id UK
JSONB gateway_response
TIMESTAMP created_at
TIMESTAMP updated_at
}

    users ||--o{ courts : "owns"
    users ||--o{ bookings : "makes"
    users ||--o{ payments : "pays"
    courts ||--o{ court_images : "has"
    courts ||--o{ bookings : "reserved via"
    bookings ||--o| payments : "paid by"

{{</mermaid>}}

#### 6.3 Double-Booking Prevention (adopted from Thanh's design)

A two-layered defense at the database level, superseding the original unconditioned constraint:

**Layer 1 — Row-level locking inside the booking transaction.** `POST /api/v1/bookings` locks the parent court row (`SELECT ... FOR UPDATE`), checks for overlapping non-cancelled bookings, then inserts with `PENDING` status — returning `409 Conflict` and rolling back if an overlap exists.

**Layer 2 — Partial exclusion constraint** as the last line of defense, even against manual SQL:

```sql
ALTER TABLE bookings
ADD CONSTRAINT chk_bookings_overlap
EXCLUDE USING gist (
  court_id WITH =,
  tsrange(start_time, end_time) WITH &&
)
WHERE (status != 'CANCELLED');
```

#### 6.4 Differences from the Individual Versions

**vs. Hieu's version (Week 2):**

| Aspect                | Hieu's version                                        | Unified version                                                                     |
| --------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| `users.password_hash` | `NOT NULL` (bcrypt)                                    | **Dropped** — credentials live exclusively in Cognito                                 |
| `users.cognito_sub`   | Nullable `UNIQUE`                                      | **`UNIQUE, NOT NULL`** — the mandatory identity link                                  |
| `users.is_active`     | Absent                                                 | **Added** (from Nguyen) — soft delete / suspension                                    |
| Exclusion constraint  | Unconditioned on `(court_id, tsrange)`                 | **Refined** with `WHERE (status != 'CANCELLED')` + paired with row-level locking      |
| API surface           | 4 payment endpoints                                    | **15 endpoints** across auth, booking, and payment                                    |
| Payment design        | —                                                      | Carried over **unchanged** (endpoints, `payments` table, 9-step flow)                 |

**vs. Nguyen's version:**

| Aspect                    | Nguyen's version                                        | Unified version                                                                                        |
| ------------------------- | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Schema size               | 9 tables                                                 | **5 tables** — 4 identity tables replaced by Cognito                                                      |
| OAuth (Google/Facebook)   | `user_identities` table                                  | **Cognito federated identity providers** (Hosted UI)                                                      |
| Sessions & forced logout  | `refresh_tokens` table with `is_revoked`                 | **Cognito refresh tokens + `GlobalSignOut`** — same capability, zero DB writes                            |
| RBAC / multi-role         | `roles` + `user_roles` bridge tables                     | **Cognito Groups** — a user in multiple groups keeps his multi-role requirement; JWT carries `cognito:groups` |
| `users.username`          | Optional column                                          | Dropped — Cognito `preferred_username` attribute                                                          |
| Auth API contracts        | 5 endpoints hitting local tables                         | **Same 5 contracts preserved**, re-implemented as FastAPI wrappers around Cognito APIs                    |
| Auth flow                 | Short-lived JWT + revokable refresh token (self-managed) | Identical flow, with **Cognito as the token authority**                                                   |

**vs. Thanh's version:**

| Aspect                     | Thanh's version                                  | Unified version                                                             |
| -------------------------- | ------------------------------------------------- | ---------------------------------------------------------------------------- |
| Booking endpoints          | 6 endpoints, input/output left unspecified        | **All 6 adopted unchanged; input/output specifications completed**            |
| Double-booking prevention  | Two-layer design (row lock + partial constraint)  | **Adopted wholesale as the canonical mechanism**                              |
| `users` table referenced   | Original Week 2 schema (`password_hash`, nullable `cognito_sub`) | Updated to the unified Cognito-centric `users` design            |
| Payment flow & lifecycle   | Mirrored from Hieu's Week 2 design                | Unchanged                                                                     |

### 7. Timeline & Milestones

| Week | Deliverable                                                   |
| ---- | ------------------------------------------------------------- |
| 1–2  | Set up VPC, EC2, RDS, Cognito; deploy FastAPI skeleton        |
| 3    | Implement EC2 Auto Scaling and ELB; integrate Cognito auth    |
| 4    | Build booking logic; enforce RDS transactional constraints    |
| 5    | Implement API Gateway + Lambda payment flow                   |
| 6    | Wire SNS notifications; connect confirmation Lambda to RDS    |
| 7    | Set up CloudWatch dashboards and alarms; S3 asset integration |
| 8    | End-to-end testing, performance tuning, final documentation   |

### 8. Budget Estimation

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

### 9. Risk Assessment

#### Risk Matrix

- **Double-booking race condition**: High impact, low probability - mitigated by RDS transactions and row-level locking.
- **EC2 instance failure**: High impact, low probability - mitigated by Auto Scaling and ELB health checks.
- **Lambda cold start latency on payment**: Medium impact, low probability - mitigated by provisioned concurrency if needed.
- **Cost overrun**: Medium impact, low probability - mitigated by CloudWatch billing alarms and Auto Scaling cooldown policies.

#### Mitigation Strategies

- Use `SELECT FOR UPDATE` or PostgreSQL advisory locks to prevent concurrent booking conflicts at the database level.
- Configure ELB health checks to automatically replace unhealthy EC2 instances.
- Set AWS budget alerts to notify when monthly spend exceeds a defined threshold.

### 10. Expected Outcomes

- **Technical**: A fully functional court booking system with real-time availability, secure payment, and instant notification - deployed on a production-grade hybrid AWS architecture.
- **Educational**: Demonstrated hands-on experience with EC2, RDS, Cognito, Lambda, API Gateway, SNS, S3, and CloudWatch within a single cohesive project.
- **Operational**: Auto Scaling ensures the system handles traffic spikes without manual intervention; CloudWatch provides ongoing visibility into system health.
