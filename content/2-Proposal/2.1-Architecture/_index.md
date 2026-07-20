---
title: "Architecture Design"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 2.1. </b> "
---

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

**Guiding principle: maximise the use of AWS managed services.** Wherever an AWS service natively covers a concern, the design delegates to it instead of rebuilding it in the database or application code. The largest consequence is in identity: Amazon Cognito is the sole credential and session authority, which keeps the relational schema at **5 core tables** (extended to **7** by the Court Manager revision — see §6.5).

| Concern                                | Handled by                                                            | Replaces (from individual designs)                     |
| -------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------- |
| Credential storage (passwords)         | Cognito User Pool                                                     | `users.password_hash` column                            |
| Social login (Google / Facebook)       | Cognito federated identity providers (Hosted UI)                      | `user_identities` table                                 |
| Session lifecycle, refresh, revocation | Cognito refresh tokens + `GlobalSignOut`                              | `refresh_tokens` table                                  |
| Roles & permissions (RBAC)             | Cognito Groups (`cognito:groups` claim in the JWT)                    | `roles` + `user_roles` bridge tables                    |
| Multi-role users (player + court_manager)| A user may belong to multiple Cognito Groups                          | Many-to-many `user_roles` design                        |

#### 6.1 Unified API Documentation (31 endpoints)

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

**Court Management — 10 endpoints** (added by the §6.5 revision; require the `court_manager` role via `cognito:groups`, plus an **ownership check** — `courts.owner_id` must equal the caller — on every court-specific route)

| # | Endpoint | Method | Input | Output | Use Case |
| - | -------- | ------ | ----- | ------ | -------- |
| 1 | `/api/v1/courts` | `POST` | `name`, `description`, `address`, `sport_type`, `price_per_hour` | `court_id`, `status: "PENDING"` | Manager registers a new court; it goes live after admin approval. |
| 2 | `/api/v1/courts/{court_id}` | `PUT` | Any of: `name`, `description`, `address`, `sport_type`, `price_per_hour` | Updated court object | Manager updates court info or the base rental fee. |
| 3 | `/api/v1/courts/{court_id}/status` | `PATCH` | `status` (`ACTIVE` / `INACTIVE` / `MAINTENANCE`) | Updated court object | Temporarily close or reopen a court. |
| 4 | `/api/v1/courts/{court_id}/images` | `POST` | `content_type` | `image_id`, `upload_url` (S3 presigned), `image_url` | Two-step photo upload: get a presigned URL, then PUT the file directly to S3. |
| 5 | `/api/v1/courts/{court_id}/images/{image_id}` | `DELETE` | — | `204 No Content` | Remove a photo (`is_primary` reassignment handled server-side). |
| 6 | `/api/v1/courts/{court_id}/schedule` | `PUT` | `[{day_of_week, open_time, close_time, price_per_hour?}]` | The saved schedule | Replace the weekly operating hours, with optional peak-pricing per window. |
| 7 | `/api/v1/courts/{court_id}/blackouts` | `POST` | `start_time`, `end_time`, `reason` | `blackout_id` — or `409` with conflicting bookings | Block a one-off period; rejected if it overlaps a CONFIRMED booking. |
| 8 | `/api/v1/courts/{court_id}/blackouts/{blackout_id}` | `DELETE` | — | `204 No Content` | Lift a closure. |
| 9 | `/api/v1/manager/bookings` | `GET` | Query: `court_id?`, `date_from`, `date_to`, `status?`, `page`, `limit` | `{ data: [...bookings + player contact], total, page, limit }` | The incoming-bookings view across all courts the manager owns. |
| 10 | `/api/v1/manager/bookings/{booking_id}` | `PATCH` | `action` (`cancel` / `complete` / `no_show`), `reason?` | Updated booking | Manager cancels (triggers the refund path via the payment Lambda + SNS notice) or closes out past bookings. |

**Payment — 4 endpoints** (from Hieu's Week 2 design, unchanged)

| # | Endpoint                         | Method | Input                                                             | Output                                                | Use Case                                                                                                                     |
| - | -------------------------------- | ------ | ------------------------------------------------------------------ | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| 1 | `/api/v1/payments`               | `POST` | `booking_id`, `amount`, `payment_method`                           | `payment_id`, `checkout_url`, `status: "PENDING"`      | Player initiates payment after confirming a booking; creates a PENDING payment record and returns a checkout URL.             |
| 2 | `/api/v1/payments/webhook`       | `POST` | `payment_id`, `transaction_id`, `status`, `gateway_data`           | `{ success: boolean }`                                 | Gateway callback — entry point of the serverless path: API Gateway → Lambda updates RDS → SNS notifies the player.            |
| 3 | `/api/v1/payments/{payment_id}`  | `GET`  | `payment_id` (path), Bearer token                                  | Full payment object                                    | Player or admin checks the status of a specific payment.                                                                       |
| 4 | `/api/v1/payments`               | `GET`  | Bearer token; `page`, `limit`                                      | `{ data: [...payments], total, page, limit }`          | Player views their paginated payment history.                                                                                  |

**Admin Operations — 3 endpoints** (added by the §6.6 revision; require the `admin` role via `cognito:groups`)

| # | Endpoint | Method | Input | Output | Use Case |
| - | -------- | ------ | ----- | ------ | -------- |
| 1 | `/api/v1/admin/courts` | `GET` | Query: `status` (default `PENDING`), `page`, `limit` | `{ data: [...courts], total, page, limit }` | Admin views the queue of newly registered courts awaiting review before they go live. |
| 2 | `/api/v1/admin/courts/{court_id}/review` | `PATCH` | `action` (`approve` / `reject`), `reason` (required on reject) | Updated court object | Admin flips `PENDING` → `ACTIVE` or `REJECTED`; the reason is persisted to `courts.rejection_reason` and an SNS notification goes to the court manager. |
| 3 | `/api/v1/admin/users/{user_id}/role` | `PATCH` | `role` (`player` / `court_manager` / `admin`) | Updated user object | Admin changes a user's role **Cognito-first**: update Cognito groups (`AdminAddUserToGroup` / `AdminRemoveUserFromGroup`), then the `users.role` cache — see §6.6. |

**Manager Analytics — 1 endpoint** (added by the §6.6 revision; requires the `court_manager` role)

| # | Endpoint | Method | Input | Output | Use Case |
| - | -------- | ------ | ----- | ------ | -------- |
| 1 | `/api/v1/manager/revenue` | `GET` | Query: `date_from`, `date_to`, `court_id?`, `group_by?` (`day` / `court`) | `{ total_revenue, currency, bookings_count, breakdown? }` | Dashboard revenue: DB-level `SUM` over **`payments` with `status = 'SUCCESS'`** joined to the caller's courts (`owner_id` scoped — IDOR rule); `group_by` powers charts. Definition rationale in §6.6. |

**User Profile — 2 endpoints** (added by the §6.6 revision; any authenticated user — implementation deferred, low priority)

| # | Endpoint | Method | Input | Output | Use Case |
| - | -------- | ------ | ----- | ------ | -------- |
| 1 | `/api/v1/users/me` | `GET` | Bearer token (header) | User profile object | Current user's profile (name, phone, avatar, roles). The `me` pattern (as in `/bookings/me`) derives identity from the JWT — no user-ID parameter to attack. |
| 2 | `/api/v1/users/me` | `PUT` | Any of: `full_name`, `phone`, `avatar_url` | Updated profile object | Update own personal information after registration. |

#### 6.2 Unified Database Design (7 tables)

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
| `role`        | `VARCHAR(20)`  | `DEFAULT 'player'`           | `player` / `admin` / `court_manager` — primary role cached for DB queries; authorization uses the `cognito:groups` JWT claim |
| `is_active`   | `BOOLEAN`      | `DEFAULT true`               | Soft delete / suspension flag                                                                 |
| `created_at`  | `TIMESTAMP`    | `DEFAULT NOW()`              | Account creation time                                                                         |
| `updated_at`  | `TIMESTAMP`    | `DEFAULT NOW()`              | Last profile update                                                                           |

> `password_hash` is intentionally absent: credentials live exclusively in Cognito. `court_images` and `payments` are carried over unchanged — full column specifications are in the [Week 2 worklog](/1-worklog/1.2-week2/). `courts` and `bookings` are modified, and two scheduling tables are added, by the Court Manager revision (§6.5):

**`courts`** (modified): `status` values become `PENDING` / `ACTIVE` / `INACTIVE` / `MAINTENANCE` / `REJECTED` — manager-registered courts default to `PENDING` until an admin approves them; public search returns only `ACTIVE`. The §6.6 revision adds `rejection_reason VARCHAR(500)` (nullable) so a rejection's reason survives beyond the SNS notification and is visible in the manager dashboard.

**`bookings`** (modified): two audit columns added for manager actions —

| Column                | Data Type | Constraints               | Description                                        |
| --------------------- | --------- | ------------------------- | --------------------------------------------------- |
| `cancelled_by`        | `UUID`    | `FK → users(id)`, nullable | Who cancelled (player, manager, or system)          |
| `cancellation_reason` | `TEXT`    |                           | Free-text reason shown to the player                |

`status` additionally gains `NO_SHOW` (set by the manager after a missed slot).

**`court_schedules`** (new) — weekly recurring operating hours; multiple windows per day allowed (e.g. 06:00–11:00 and 16:00–22:00):

| Column           | Data Type       | Constraints                        | Description                                                             |
| ---------------- | --------------- | ---------------------------------- | ------------------------------------------------------------------------ |
| `id`             | `UUID`          | `PRIMARY KEY`                      |                                                                          |
| `court_id`       | `UUID`          | `NOT NULL, FK → courts(id)`        | Parent court                                                             |
| `day_of_week`    | `SMALLINT`      | `NOT NULL, CHECK (0–6)`            | 0 = Monday                                                               |
| `open_time`      | `TIME`          | `NOT NULL`                         | Window start                                                             |
| `close_time`     | `TIME`          | `NOT NULL, CHECK (> open_time)`    | Window end                                                               |
| `price_per_hour` | `DECIMAL(12,2)` | nullable                           | Peak-pricing override; `NULL` falls back to `courts.price_per_hour`      |

**`court_blackouts`** (new) — one-off closures (maintenance, private events):

| Column       | Data Type      | Constraints                 | Description       |
| ------------ | -------------- | --------------------------- | ------------------ |
| `id`         | `UUID`         | `PRIMARY KEY`               |                    |
| `court_id`   | `UUID`         | `NOT NULL, FK → courts(id)` | Parent court       |
| `start_time` | `TIMESTAMP`    | `NOT NULL`                  | Closure start      |
| `end_time`   | `TIMESTAMP`    | `NOT NULL`                  | Closure end        |
| `reason`     | `VARCHAR(255)` |                             | Shown to players   |

**Availability rule:** a slot is bookable iff it lies inside a `court_schedules` window **and** does not overlap a `court_blackouts` row **and** no overlapping non-cancelled booking exists (the row lock + exclusion constraint in §6.3 still guard the last condition).

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
VARCHAR rejection_reason
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
court_schedules {
UUID id PK
UUID court_id FK
SMALLINT day_of_week
TIME open_time
TIME close_time
DECIMAL price_per_hour
}
court_blackouts {
UUID id PK
UUID court_id FK
TIMESTAMP start_time
TIMESTAMP end_time
VARCHAR reason
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
UUID cancelled_by FK
TEXT cancellation_reason
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

    users ||--o{ courts : "manages"
    users ||--o{ bookings : "makes"
    users ||--o{ payments : "pays"
    courts ||--o{ court_images : "has"
    courts ||--o{ court_schedules : "operates on"
    courts ||--o{ court_blackouts : "closed by"
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
| API surface           | 4 payment endpoints                                    | **31 endpoints** across auth, booking, court management, payment, admin, and analytics |
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

#### 6.5 Design Revision (07/14/2026) — Court Manager Role

The team identified a missing actor: the person who runs a court on the platform. The role existed nominally in the original schema (`court_owner` + `courts.owner_id`), but had **no API surface and no schedule model** — availability was derived solely from bookings, meaning a court had no opening hours and no way to close for maintenance.

**Role name.** Renamed `court_owner` → **`court_manager`**: the person operating a court is often not its legal owner (staff, hired managers). Alternatives considered: `host` (friendly but vague), `partner` (marketplace-style, revisit if venues onboard as businesses), `venue_manager` (heavier, fits only if the platform expands beyond courts). Migration cost was near zero — a Cognito group name plus a default string in `users.role`.

**Capabilities added** (endpoints in §6.1, schema in §6.2): register a court (admin-approved via the `PENDING` status), update court info / rental fee / photos, define weekly operating hours with optional peak pricing (`court_schedules`), declare one-off closures (`court_blackouts`), and handle incoming bookings (view across owned courts, cancel with audited reason, mark `COMPLETED` / `NO_SHOW`).

**Two deliberate design decisions:**

1. **No manager-approval gate on bookings.** Payment remains the confirmation mechanism, so the existing serverless payment flow is untouched. The manager *handles* bookings through visibility and cancellation, not a manual accept step — an accept-before-pay gate would complicate the payment flow for little benefit at this scale.
2. **Ownership enforced in queries, not just role checks.** Every manager endpoint filters or validates `courts.owner_id` against the caller, so manager A can never read or modify manager B's courts — the standard defense against IDOR (Insecure Direct Object Reference).

**Ripple effects:** Cognito group `court_manager` replaces `court_owner`; one Alembic migration (2 new tables, `courts.status` values, 2 `bookings` columns); `GET /courts/{id}/availability` now merges schedule + blackouts + bookings; `POST /bookings` additionally validates the slot lies inside operating hours; the frontend gains a manager dashboard section.

#### 6.6 Design Revision (07/19/2026) — Admin Operations & Manager Analytics

Proposed by **Nguyen** and reviewed in the Week 5 team meeting: six endpoints extending §6.5, closing two gaps it left open. Adopted with the adjustments below; API grows 25 → **31 endpoints**.

**Gap 1 — the approval loop had no driver.** §6.5 made manager-registered courts start at `PENDING`, but no endpoint could move them out of it: managers can only toggle approved courts, and players never see non-`ACTIVE` ones. The admin queue + review endpoints complete the state machine, with an SNS notification to the manager on every decision. The rejection reason is **persisted** (`courts.rejection_reason`) rather than living only in the notification — anything a user may need to re-read later belongs in a table, not a message.

**Gap 2 — the manager dashboard had no money API.** The revenue endpoint aggregates from **`payments` with `status = 'SUCCESS'`**, not from booking totals, because the two answer different questions — *what was priced* vs. *what money is actually held*:

| Scenario | Booking total says | Money reality | `payments`-based SUM |
| --- | --- | --- | --- |
| Confirmed and played (`COMPLETED`) | counted | held ✅ | counted ✅ |
| `NO_SHOW` — paid, never came | counted | held ✅ | counted ✅ |
| `CANCELLED`, refunded | counted ❌ | returned | excluded ✅ (`REFUNDED`) |
| `PENDING`, never paid | counted ❌ | never existed | excluded ✅ |

Refund-handling falls out *by construction* — no special cases. Scoping follows the §6.5 ownership rule (join through `courts.owner_id = caller`), and `group_by=day|court` lets one endpoint serve both the headline number and charts, keeping aggregation in Postgres.

**Role changes are Cognito-first.** `users.role` is a cache; the authority is Cognito groups (§6). The role endpoint must therefore call `AdminAddUserToGroup`/`AdminRemoveUserFromGroup` **before** updating the DB row — a DB-only update would change the UI label while every authorization check still reads the old JWT claim. Note: already-issued JWTs keep their old `cognito:groups` until refreshed (~15 min), so a promotion takes effect on next login/refresh — expected behavior, not a bug.

**Naming standardization.** Profile endpoints use `GET/PUT /users/me` (matching `/bookings/me`) instead of verb-style URLs: resources in the path, verbs in the method — and the `me` pattern removes the user-ID parameter entirely, eliminating that IDOR surface. Profile implementation is **deferred** (low priority).

**Ripple effects:** one Alembic migration (`courts.rejection_reason`, nullable — backwards-compatible); ERD updated (Mermaid + drawio); the frontend gains an admin review screen and the manager dashboard's revenue chart; role-change requires the backend's IAM/SDK access to Cognito admin APIs.

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

---

### 11. Glossary

| Abbreviation | Meaning |
| --- | --- |
| API | Application Programming Interface — the contract through which software components communicate |
| AWS | Amazon Web Services — Amazon's cloud computing platform |
| AZ | Availability Zone — an isolated data-center cluster within an AWS Region |
| CDN | Content Delivery Network — geographically distributed cache serving static content close to users |
| CI/CD | Continuous Integration / Continuous Delivery — automated building, testing, and deployment |
| DB | Database |
| EC2 | Amazon Elastic Compute Cloud — virtual servers on AWS |
| ELB | Elastic Load Balancer — distributes incoming traffic across instances |
| FK | Foreign Key — a column referencing another table's primary key |
| HTTP | HyperText Transfer Protocol |
| IDOR | Insecure Direct Object Reference — accessing another user's resources by manipulating object IDs |
| JSONB | JSON Binary — PostgreSQL's binary JSON column type |
| JWT | JSON Web Token — signed token carrying identity claims |
| MAU | Monthly Active Users |
| OAuth | Open Authorization — standard for delegated third-party login |
| PK | Primary Key — the column uniquely identifying each row |
| RBAC | Role-Based Access Control |
| RDS | Amazon Relational Database Service — managed SQL databases |
| REST | Representational State Transfer — architectural style for HTTP APIs |
| S3 | Amazon Simple Storage Service — object storage |
| SES | Amazon Simple Email Service — transactional email sending |
| SNS | Amazon Simple Notification Service — pub/sub messaging and notifications |
| SQL | Structured Query Language |
| UI | User Interface |
| UK | Unique Key — column(s) whose values must be unique |
| URL | Uniform Resource Locator — web address |
| UUID | Universally Unique Identifier |
| VPC | Amazon Virtual Private Cloud — an isolated virtual network on AWS |
