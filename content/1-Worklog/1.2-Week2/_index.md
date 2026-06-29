---
title: "Week 2 Worklog"
date: 2026-06-22
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Week 2 Objectives:

- Study AWS services in depth using AWS Skill Builder as preparation for the AWS Developer Associate certification.
- Complete Payment API documentation and full database design for the court booking application.
- Present payment logic and DB design to the team; review UI draft.

### Tasks to be carried out this week:

| Day | Task                                                                                                                                                                                                                                                             | Start Date | Completion Date | Reference Material                                                                    |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | --------------- | ------------------------------------------------------------------------------------- |
| 2   | - **AWS Skill Builder — Official Practice Question Set: AWS Certified Developer – Associate (DVA-C02)** <br> - Review court booking architecture; plan week deliverables                                                                                         | 06/22/2026 | 06/22/2026      | [AWS Skill Builder](https://skillsprofile.skillbuilder.aws/user/minervaph/cloudquest) |
| 4   | - **AWS Skill Builder — Cloud Quest: Cloud Practitioner** <br>&emsp; + Cloud Computing Essentials <br>&emsp; + Cloud First Steps <br>&emsp; + Computing Solutions (AI mode) <br> - **AWS Escape Room**: Exam Prep for AWS Certified Cloud Practitioner (CLF-C02) | 06/24/2026 | 06/24/2026      | [AWS Skill Builder](https://skillsprofile.skillbuilder.aws/user/minervaph/cloudquest) |
| 5   | - Complete Payment API documentation (4 endpoints) <br> - Design `payments` table schema                                                                                                                                                                         | 06/25/2026 | 06/25/2026      |                                                                                       |
| 6   | - **AWS Skill Builder — Cloud Quest: Cloud Practitioner** <br>&emsp; + Networking Concepts <br>&emsp; + Computing Solutions <br> - Complete full database design (5 tables + ERD) <br> - Document end-to-end payment flow                                        | 06/26/2026 | 06/26/2026      | [AWS Skill Builder](https://skillsprofile.skillbuilder.aws/user/minervaph/cloudquest) |
| 7   | - **Team meeting (06/27/2026)**: present payment logic & DB design <br> - Review UI draft shared by Danh                                                                                                                                                         | 06/27/2026 | 06/27/2026      |                                                                                       |

### Week 2 Achievements:

**AWS Skill Builder — Official Practice Question Set: AWS Certified Developer – Associate (DVA-C02)**

Completed on **06/22/2026** — the official exam-style practice question set for the DVA-C02 certification, covering core AWS Developer Associate topics including Lambda, API Gateway, DynamoDB, SQS, SNS, Cognito, and deployment services.

[Download Completion Certificate](/images/1-Worklog/1.2-Week2/CompletionCert-OfficialPracticeQuestionSet-DVA-C02-20260622.pdf)

---

**AWS Skill Builder — Cloud Quest: Cloud Practitioner**

| Solution                   | Completed                         | Description                                                                                                                                             |
| -------------------------- | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cloud Computing Essentials | 06/24/2026                        | Definition of cloud computing, comparison with traditional on-premise infrastructure, and value proposition. Lab: enable static website hosting on S3.  |
| Cloud First Steps          | 06/24/2026                        | AWS Global Infrastructure — Regions, Availability Zones, data centers, and Points of Presence. Lab: deploy EC2 instances across two AZs.                |
| Computing Solutions        | 06/24/2026 (AI mode) / 06/26/2026 | Overview of AWS compute services. Lab: filter EC2 instance types, connect via Session Manager, resize an instance.                                      |
| Networking Concepts        | 06/26/2026                        | Overview of networking on AWS — VPCs, subnets, route tables, internet gateways, and security groups. Lab: configure VPC networking for a web + DB tier. |

Reputation points earned:

| Service    | Points |
| ---------- | ------ |
| Amazon EC2 | 34     |
| Amazon VPC | 10     |
| AWS IAM    | 1      |
| Amazon S3  | 14     |

**Completion Receipts**

_Cloud Computing Essentials — 06/24/2026_

![Cloud Computing Essentials completion receipt](/images/1-Worklog/1.2-Week2/CloudComputingEssentials_20260624092947.png)

_Cloud First Steps — 06/24/2026_

![Cloud First Steps completion receipt](/images/1-Worklog/1.2-Week2/CloudFirstSteps_20260624103147.png)

_Computing Solutions — 06/24/2026 (AI mode)_

![Computing Solutions AI mode completion receipt](/images/1-Worklog/1.2-Week2/ComputingSolutions_AI-mode_20260624154735.png)

_Computing Solutions — 06/26/2026_

![Computing Solutions completion receipt](/images/1-Worklog/1.2-Week2/ComputingSolutions_20260626163021.png)

_Networking Concepts — 06/26/2026 (AI mode)_

![Networking Concepts AI mode completion receipt](/images/1-Worklog/1.2-Week2/NetworkingConcepts_AI-mode_20260626154612.png)

_Networking Concepts — 06/26/2026_

![Networking Concepts completion receipt](/images/1-Worklog/1.2-Week2/NetworkingConcepts_20260626161508.png)

**AWS Escape Room: Exam Prep for AWS Certified Cloud Practitioner (CLF-C02)**

Completed the single-player practice mode on **06/24/2026** — a 2–3 hour 3D virtual escape room covering exam-style questions and hands-on exercises across: Amazon API Gateway, Amazon DynamoDB, Amazon EC2, AWS IAM, AWS Lambda, and Amazon S3.

[Download Completion Certificate](/images/1-Worklog/1.2-Week2/CompletionCert-EscapeRoom.pdf)

**Payment Feature Deliverable**

- Designed and documented 4 Payment API endpoints (initiate, webhook, get by ID, list history).
- Designed the `payments` table schema with 11 columns covering identity, relations, status lifecycle, and gateway auditing.
- Completed the full 5-table database design (`users`, `courts`, `court_images`, `bookings`, `payments`) with ERD.
- Documented the 9-step end-to-end payment flow mapping each action to DB operations and AWS services.

---

### Team Meeting — 06/27/2026

**Attendees:** Hieu, Thanh, Nguyen, Hung
**Absent:** Danh

**Presentations**

- **Hieu** demonstrated the end-to-end payment logic and walked the team through the proposed database design for the application.
- **Danh** (submitted asynchronously prior to the meeting) shared an initial UI draft for review.

**Task Distribution**

| Track                         | Owner       | Deliverable                                                                                                  |
| ----------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------ |
| Frontend — UI Design          | Danh & Hung | Unify and improve the UI design based on Danh's initial draft                                                |
| Frontend — Tech Stack         | Danh & Hung | Propose tech stack for frontend implementation                                                               |
| Backend — Tech Stack          | All BE      | Propose tech stack for backend implementation                                                                |
| Backend — Architecture        | Hieu        | Integrate Amazon Amplify for frontend hosting into the architecture; redraw the updated architecture diagram |
| Backend — Booking Management  | Thanh       | API documentation (endpoint name, input, output, use case) and DB design (tables, columns, data types)       |
| Backend — User Authentication | Nguyen      | API documentation (endpoint name, input, output, use case) and DB design (tables, columns, data types)       |

---

### Hieu's Deliverable — Payment Feature

#### API Documentation

| #   | Endpoint                        | Method | Input                                                                                      | Output                                                                                                                     | Use Case                                                                                                                                                                                                                                     |
| --- | ------------------------------- | ------ | ------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `/api/v1/payments`              | `POST` | `booking_id` (UUID), `amount` (decimal), `payment_method` (string)                         | `payment_id` (UUID), `checkout_url` (string), `status: "PENDING"`                                                          | Player initiates payment after confirming a court booking. Creates a PENDING payment record in RDS and returns a checkout URL to redirect the player.                                                                                        |
| 2   | `/api/v1/payments/webhook`      | `POST` | `payment_id` (UUID), `transaction_id` (string), `status` (string), `gateway_data` (object) | `{ success: boolean }`                                                                                                     | Payment gateway notifies the system of the transaction result. This is the entry point of the serverless path: API Gateway receives the callback → invokes Lambda to update payment status in RDS → triggers SNS notification to the player. |
| 3   | `/api/v1/payments/{payment_id}` | `GET`  | `payment_id` (path param), Bearer token (header)                                           | `payment_id`, `booking_id`, `amount`, `currency`, `status`, `payment_method`, `transaction_id`, `created_at`, `updated_at` | Player or admin checks the status of a specific payment.                                                                                                                                                                                     |
| 4   | `/api/v1/payments`              | `GET`  | Bearer token (header), `page` (int), `limit` (int)                                         | `{ data: [...payments], total, page, limit }`                                                                              | Player views their full payment history with pagination.                                                                                                                                                                                     |

#### Database Design

**Table: `payments`**

| Column             | Data Type        | Constraints                     | Description                                                |
| ------------------ | ---------------- | ------------------------------- | ---------------------------------------------------------- |
| `id`               | `UUID`           | `PRIMARY KEY`                   | Unique payment identifier                                  |
| `booking_id`       | `UUID`           | `NOT NULL`, `FK → bookings(id)` | The booking this payment is for                            |
| `user_id`          | `UUID`           | `NOT NULL`, `FK → users(id)`    | Player who initiated the payment                           |
| `amount`           | `DECIMAL(12, 2)` | `NOT NULL`                      | Payment amount                                             |
| `currency`         | `VARCHAR(3)`     | `NOT NULL`, `DEFAULT 'VND'`     | Currency code                                              |
| `status`           | `VARCHAR(20)`    | `NOT NULL`, `DEFAULT 'PENDING'` | `PENDING` / `SUCCESS` / `FAILED` / `REFUNDED`              |
| `payment_method`   | `VARCHAR(50)`    | `NOT NULL`                      | `CREDIT_CARD` / `BANK_TRANSFER` / `E_WALLET`               |
| `transaction_id`   | `VARCHAR(255)`   | `UNIQUE`                        | Transaction ID returned by the payment gateway             |
| `gateway_response` | `JSONB`          |                                 | Raw response payload from the payment gateway for auditing |
| `created_at`       | `TIMESTAMP`      | `NOT NULL`, `DEFAULT NOW()`     | Payment initiation time                                    |
| `updated_at`       | `TIMESTAMP`      | `NOT NULL`, `DEFAULT NOW()`     | Time of last status update                                 |

**Notes:**

- `transaction_id` is nullable until the gateway responds (stays `NULL` while status is `PENDING`).
- `gateway_response` stores the full raw payload from the gateway — useful for dispute resolution and debugging without needing to re-query the gateway.
- The `status` column is updated by the confirmation Lambda (webhook path), not by the EC2 monolith, keeping the two paths clearly separated.

---

### Full Database Design

#### Tables

**`users`**

| Column          | Data Type      | Constraints        | Description                                                   |
| --------------- | -------------- | ------------------ | ------------------------------------------------------------- |
| `id`            | `UUID`         | `PRIMARY KEY`      | Unique user identifier                                        |
| `email`         | `VARCHAR(255)` | `UNIQUE, NOT NULL` | Login email                                                   |
| `password_hash` | `VARCHAR(255)` | `NOT NULL`         | Bcrypt-hashed password                                        |
| `full_name`     | `VARCHAR(255)` | `NOT NULL`         | Display name                                                  |
| `phone`         | `VARCHAR(20)`  |                    | Contact number                                                |
| `cognito_sub`   | `VARCHAR(255)` | `UNIQUE`           | Cognito User Pool subject ID — links JWT to local user record |
| `role`          | `VARCHAR(20)`  | `DEFAULT 'player'` | `player` / `admin` / `court_owner`                            |
| `avatar_url`    | `VARCHAR(500)` |                    | S3 URL of profile photo                                       |
| `created_at`    | `TIMESTAMP`    | `DEFAULT NOW()`    | Account creation time                                         |
| `updated_at`    | `TIMESTAMP`    | `DEFAULT NOW()`    | Last profile update                                           |

**`courts`**

| Column           | Data Type       | Constraints        | Description                                        |
| ---------------- | --------------- | ------------------ | -------------------------------------------------- |
| `id`             | `UUID`          | `PRIMARY KEY`      | Unique court identifier                            |
| `owner_id`       | `UUID`          | `FK → users(id)`   | Court owner (user with role `court_owner`)         |
| `name`           | `VARCHAR(255)`  | `NOT NULL`         | Court display name                                 |
| `description`    | `TEXT`          |                    | Court details                                      |
| `address`        | `VARCHAR(500)`  | `NOT NULL`         | Physical location                                  |
| `sport_type`     | `VARCHAR(50)`   | `NOT NULL`         | `badminton` / `tennis` / `football` / `basketball` |
| `price_per_hour` | `DECIMAL(12,2)` | `NOT NULL`         | Hourly rate in VND                                 |
| `status`         | `VARCHAR(20)`   | `DEFAULT 'ACTIVE'` | `ACTIVE` / `INACTIVE` / `MAINTENANCE`              |
| `created_at`     | `TIMESTAMP`     | `DEFAULT NOW()`    |                                                    |
| `updated_at`     | `TIMESTAMP`     | `DEFAULT NOW()`    |                                                    |

**`court_images`**

| Column       | Data Type      | Constraints                 | Description                     |
| ------------ | -------------- | --------------------------- | ------------------------------- |
| `id`         | `UUID`         | `PRIMARY KEY`               |                                 |
| `court_id`   | `UUID`         | `NOT NULL, FK → courts(id)` | Parent court                    |
| `image_url`  | `VARCHAR(500)` | `NOT NULL`                  | S3 URL of the photo             |
| `is_primary` | `BOOLEAN`      | `DEFAULT false`             | Whether this is the cover photo |
| `created_at` | `TIMESTAMP`    | `DEFAULT NOW()`             |                                 |

**`bookings`**

| Column         | Data Type       | Constraints                 | Description                                         |
| -------------- | --------------- | --------------------------- | --------------------------------------------------- |
| `id`           | `UUID`          | `PRIMARY KEY`               | Unique booking identifier                           |
| `user_id`      | `UUID`          | `NOT NULL, FK → users(id)`  | Player who made the booking                         |
| `court_id`     | `UUID`          | `NOT NULL, FK → courts(id)` | Court being reserved                                |
| `start_time`   | `TIMESTAMP`     | `NOT NULL`                  | Slot start                                          |
| `end_time`     | `TIMESTAMP`     | `NOT NULL`                  | Slot end                                            |
| `total_amount` | `DECIMAL(12,2)` | `NOT NULL`                  | Price at time of booking                            |
| `status`       | `VARCHAR(20)`   | `DEFAULT 'PENDING'`         | `PENDING` / `CONFIRMED` / `CANCELLED` / `COMPLETED` |
| `note`         | `TEXT`          |                             | Optional player note                                |
| `created_at`   | `TIMESTAMP`     | `DEFAULT NOW()`             |                                                     |
| `updated_at`   | `TIMESTAMP`     | `DEFAULT NOW()`             |                                                     |

> Double-booking prevention: add a PostgreSQL exclusion constraint using `btree_gist` on `(court_id, tsrange(start_time, end_time))` to reject overlapping time ranges at the DB level.

**`payments`** — see Hieu's Deliverable above.

#### ERD

{{<mermaid>}}
erDiagram
users {
UUID id PK
VARCHAR email UK
VARCHAR password_hash
VARCHAR full_name
VARCHAR phone
VARCHAR cognito_sub UK
VARCHAR role
VARCHAR avatar_url
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

---

### Payment Flow

The payment feature spans both deployment paths. The table below maps each step to the actor, DB operation, and AWS service involved.

| Step | Actor                    | Action                                                  | DB Operation                                                                           | AWS Service        |
| ---- | ------------------------ | ------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------ |
| 1    | Player                   | Searches and selects a court + time slot                | `SELECT` from `courts`, check no overlap in `bookings`                                 | EC2 (FastAPI)      |
| 2    | Player                   | Confirms booking                                        | `INSERT` into `bookings` with `status = 'PENDING'`                                     | EC2 (FastAPI)      |
| 3    | Player                   | Initiates payment                                       | `INSERT` into `payments` with `status = 'PENDING'`, returns `checkout_url`             | EC2 (FastAPI)      |
| 4    | Player                   | Redirected to payment gateway and completes transaction | —                                                                                      | External gateway   |
| 5    | Gateway                  | Sends webhook callback                                  | —                                                                                      | Amazon API Gateway |
| 6    | Lambda (Process Payment) | Validates webhook, updates payment record               | `UPDATE payments SET status = 'SUCCESS', transaction_id = ..., gateway_response = ...` | AWS Lambda         |
| 7    | Lambda (Confirm Booking) | Confirms the booking                                    | `UPDATE bookings SET status = 'CONFIRMED'`                                             | AWS Lambda         |
| 8    | SNS                      | Notifies player                                         | —                                                                                      | Amazon SNS         |
| 9    | Player                   | Views confirmation                                      | `SELECT` from `bookings` JOIN `payments`                                               | EC2 (FastAPI)      |

**On payment failure (Step 6 returns `FAILED`):**

- `payments.status` → `FAILED`
- `bookings.status` → `CANCELLED`
- SNS notifies the player of the failure

**State lifecycle:**

```
bookings.status:   PENDING ──► CONFIRMED ──► COMPLETED
                       └──────► CANCELLED

payments.status:   PENDING ──► SUCCESS
                       └──────► FAILED ──► (retry creates new payment record)
```

**Key design rule:** Steps 1–3 are handled by the EC2 monolith (writes to `bookings` and `payments`). Steps 5–8 are handled entirely by the serverless path (Lambda reads and writes the same RDS tables). The two paths never write to the same record at the same time — the monolith creates the records, Lambda updates them — so there is no write conflict.
