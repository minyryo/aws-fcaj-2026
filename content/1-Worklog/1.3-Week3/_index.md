---
title: "Week 3 Worklog"
date: 2026-06-29
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Week 3 Objectives:

### Tasks to be carried out this week:

| Day | Task | Start Date | Completion Date | Reference Material |
| --- | ---- | ---------- | --------------- | ------------------ |

### Week 3 Achievements:


### ERD Design Objectives:

- Design a highly normalized relational database schema to support the court booking application's core business logic.
- Implement a flexible Role-Based Access Control (RBAC) structure capable of supporting `admin`, `user`, and `court_owner` roles.
- Architect an identity management foundation that seamlessly integrates local password authentication with multi-provider OAuth (Google, Facebook).
- Establish a secure, stateful session management system using refresh tokens to enable automatic expiration and forced remote logouts.


### Tasks to be carried out for schema design:

| Day | Task | Start Date | Completion Date | Reference Material |
| :--- | :--- | :--- | :--- | :--- |
| 1 | - **Core Business Logic Design** <br>&emsp; + Draft `courts`, `court_images`, and `bookings` tables.<br>&emsp; + Define PostgreSQL exclusion constraints for double-booking prevention. | 06/29/2026 | 06/29/2026 | Application Requirements |
| 2 | - **Identity & Access Management (IAM)** <br>&emsp; + Refactor `users` table for OAuth compatibility.<br>&emsp; + Design `user_identities` table for multi-provider linking.<br>&emsp; + Implement `refresh_tokens` table for secure session invalidation. | 06/30/2026 | 06/30/2026 | OAuth 2.0 / JWT Best Practices |
| 3 | - **Role-Based Access Control (RBAC)** <br>&emsp; + Abstract roles into `roles` and `user_roles` bridge tables.<br> - **Payments Integration**<br>&emsp; + Finalize `payments` schema including gateway webhooks and JSONB auditing. | 07/01/2026 | 07/01/2026 | Payment Gateway Documentation |
| 4 | - **Schema Validation & Export** <br>&emsp; + Compile DBML code.<br>&emsp; + Generate and export the visual ERD via dbdiagram.io.<br>&emsp; + Finalize architectural documentation. | 07/02/2026 | 07/02/2026 | [dbdiagram.io](https://dbdiagram.io) |

### Design Achievements:

**Entity-Relationship Diagram (ERD) Generation**

Compiled the application's DBML specification and generated the foundational ERD. The schema resolves previous limitations by decoupling authentication methods from the core user profile and implementing a scalable many-to-many role architecture.

![Court Booking Database Schema](/images/1-Worklog/1.3-Week3/CourtBookingDraft.png)


[View full diagram with details here](https://dbdiagram.io/d/Court_booking-6a4750fb4ac62e474c1f40ec)

---

**Schema Module Breakdown**

| Module | Tables | Architectural Highlights |
| :--- | :--- | :--- |
| **Identity & Security** | `users`, `user_identities`, `refresh_tokens` | The `password_hash` is now nullable to gracefully handle OAuth-only users. The `user_identities` table allows players to link multiple external accounts (Google, Facebook) to a single internal UUID. Session lifecycles are strictly controlled via `refresh_tokens`, allowing the system to force-logout users upon token expiration or manual revocation. |
| **Access Control** | `roles`, `user_roles` | Transitioned from a static string column to a dynamic Many-to-Many RBAC structure. This allows a single user to hold multiple roles simultaneously (e.g., both a `player` and a `court_owner`), providing the backend API with granular permission validation checks. |
| **Core Operations** | `courts`, `court_images`, `bookings` | `courts` are directly tied to owners via the `owner_id` foreign key. Time-slot integrity for `bookings` is maintained at the database level using `tsrange` and `btree_gist` exclusion constraints to mathematically prevent overlapping reservations on the same court. |
| **Financials** | `payments` | Separated from the booking lifecycle to allow asynchronous webhook updates. Includes a `gateway_response` column utilizing PostgreSQL's `JSONB` format to store raw, unstructured payment provider payloads for secure auditing and dispute resolution. |

---

### DBML Source Code

Below is the Database Markup Language (DBML) code used to generate the application's ERD. This can be imported directly into [dbdiagram.io](https://dbdiagram.io) for future modifications.

```
// -- USER IDENTITY & SECURITY --

Table users {
  id UUID [primary key]
  username varchar(50) [unique, note: 'Nullable, for username registration']
  email varchar(255) [unique, not null]
  password_hash varchar(255) [note: 'Nullable for OAuth-only users']
  full_name varchar(255) [not null]
  phone varchar(20)
  avatar_url varchar(500)
  is_active boolean [default: true, note: 'Soft delete / suspension flag']
  created_at timestamp [default: `now()`]
  updated_at timestamp [default: `now()`]
}

Table user_identities {
  id UUID [primary key]
  user_id UUID [not null]
  provider varchar(50) [not null, note: 'local, google, facebook']
  provider_id varchar(255) [not null]
  created_at timestamp [default: `now()`]

  indexes {
    (provider, provider_id) [unique]
  }
}

Table refresh_tokens {
  id UUID [primary key]
  user_id UUID [not null]
  token_hash varchar(255) [unique, not null]
  device_info varchar(255)
  ip_address varchar(45)
  expires_at timestamp [not null]
  is_revoked boolean [default: false]
  created_at timestamp [default: `now()`]
}

// -- ROLE-BASED ACCESS CONTROL (RBAC) --

Table roles {
  id UUID [primary key]
  name varchar(50) [unique, not null, note: 'admin, user, court_owner']
  description text
}

Table user_roles {
  user_id UUID [not null]
  role_id UUID [not null]
  granted_at timestamp [default: `now()`]

  indexes {
    (user_id, role_id) [pk]
  }
}

// -- CORE BUSINESS LOGIC --

Table courts {
  id UUID [primary key]
  owner_id UUID [not null, note: 'Must have court_owner role']
  name varchar(255) [not null]
  description text
  address varchar(500) [not null]
  sport_type varchar(50) [not null]
  price_per_hour decimal(12,2) [not null]
  status varchar(20) [default: 'ACTIVE']
  created_at timestamp [default: `now()`]
  updated_at timestamp [default: `now()`]
}

Table court_images {
  id UUID [primary key]
  court_id UUID [not null]
  image_url varchar(500) [not null]
  is_primary boolean [default: false]
  created_at timestamp [default: `now()`]
}

Table bookings {
  id UUID [primary key]
  user_id UUID [not null]
  court_id UUID [not null]
  start_time timestamp [not null]
  end_time timestamp [not null]
  total_amount decimal(12,2) [not null]
  status varchar(20) [default: 'PENDING']
  note text
  created_at timestamp [default: `now()`]
  updated_at timestamp [default: `now()`]
}

Table payments {
  id UUID [primary key]
  booking_id UUID [not null]
  user_id UUID [not null]
  amount decimal(12,2) [not null]
  currency varchar(3) [not null, default: 'VND']
  status varchar(20) [not null, default: 'PENDING']
  payment_method varchar(50) [not null]
  transaction_id varchar(255) [unique]
  gateway_response jsonb
  created_at timestamp [not null, default: `now()`]
  updated_at timestamp [not null, default: `now()`]
}

// -- RELATIONSHIPS --

Ref: user_identities.user_id > users.id
Ref: refresh_tokens.user_id > users.id
Ref: user_roles.user_id > users.id
Ref: user_roles.role_id > roles.id
Ref: courts.owner_id > users.id
Ref: court_images.court_id > courts.id
Ref: bookings.user_id > users.id
Ref: bookings.court_id > courts.id
Ref: payments.booking_id > bookings.id
Ref: payments.user_id > users.id
```

#### Authentication API Documentation

<table style="width:100%; table-layout:fixed; word-break:break-word;">
  <colgroup>
    <col style="width:3%">
    <col style="width:18%">
    <col style="width:5%">
    <col style="width:18%">
    <col style="width:20%">
    <col style="width:36%">
  </colgroup>
  <thead>
    <tr>
      <th>#</th>
      <th>Endpoint</th>
      <th>Method</th>
      <th>Input</th>
      <th>Output</th>
      <th>Use Case</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td><code>/api/v1/auth/register</code></td>
      <td><code>POST</code></td>
      <td><code>email</code>, <code>password</code>, <code>full_name</code>, <code>username</code> (optional)</td>
      <td><code>user_id</code>, <code>message: "Success"</code></td>
      <td>User registers a new local account. Creates a record in the <code>users</code> table with a bcrypt-hashed password.</td>
    </tr>
    <tr>
      <td>2</td>
      <td><code>/api/v1/auth/login</code></td>
      <td><code>POST</code></td>
      <td><code>email_or_username</code>, <code>password</code></td>
      <td><code>access_token</code> (JWT), <code>refresh_token</code>, <code>user_data</code></td>
      <td>Standard login. Validates credentials, generates a stateless short-lived JWT, and creates an active session record in the <code>refresh_tokens</code> DB table.</td>
    </tr>
    <tr>
      <td>3</td>
      <td><code>/api/v1/auth/oauth</code></td>
      <td><code>POST</code></td>
      <td><code>provider</code> (string), <code>provider_token</code> (string)</td>
      <td><code>access_token</code> (JWT), <code>refresh_token</code>, <code>user_data</code></td>
      <td>OAuth flow (Google/Facebook). Backend verifies the provider token, upserts the <code>users</code> and <code>user_identities</code> tables, and issues standard app tokens.</td>
    </tr>
    <tr>
      <td>4</td>
      <td><code>/api/v1/auth/refresh</code></td>
      <td><code>POST</code></td>
      <td><code>refresh_token</code> (string)</td>
      <td><code>access_token</code> (new JWT)</td>
      <td>Access token expires (e.g., after 15 mins). Client sends the refresh token. Backend verifies it is not expired or revoked in <code>refresh_tokens</code>, then issues a new JWT.</td>
    </tr>
    <tr>
      <td>5</td>
      <td><code>/api/v1/auth/logout</code></td>
      <td><code>POST</code></td>
      <td>Bearer token (header)</td>
      <td><code>204 No Content</code></td>
      <td>User logs out. Backend sets <code>is_revoked = true</code> in the <code>refresh_tokens</code> table, instantly killing the session.</td>
    </tr>
  </tbody>
</table>

---

### Authentication Flow

The authentication architecture ensures short-lived access with long-lived, securely revokable sessions. The table below maps each step of the login, authorization, and auto-logout cycle.

| Step | Actor          | Action                                                  | DB Operation                                                                           | App Component      |
| ---- | -------------- | ------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------ |
| 1    | User           | Submits login credentials (or OAuth token)              | `SELECT` from `users` (and `user_roles`), verify hash                                  | EC2 (Backend)      |
| 2    | Backend        | Establishes stateful session                            | `INSERT` into `refresh_tokens` with `expires_at` and `device_info`                     | EC2 (Backend)      |
| 3    | Backend        | Returns JWT payload                                     | —                                                                                      | EC2 (Backend)      |
| 4    | User           | Accesses protected route (e.g., Book a court)           | — (Stateless check: middleware verifies JWT signature and extracts embedded roles)     | API Middleware     |
| 5    | User / System  | Access Token expires (e.g., after 15 minutes)           | —                                                                                      | API Middleware     |
| 6    | User (Frontend)| Silently calls `/api/v1/auth/refresh` in background     | `SELECT` from `refresh_tokens` ensuring `is_revoked = false` and not expired           | EC2 (Backend)      |
| 7    | Backend        | Issues new Access Token (resumes session)               | —                                                                                      | EC2 (Backend)      |
| 8    | User           | Clicks "Logout" manually                                | `UPDATE refresh_tokens SET is_revoked = true`                                          | EC2 (Backend)      |

**On compromised account / Admin forced logout:**
- Admin hits an internal endpoint or the user changes their password.
- `UPDATE refresh_tokens SET is_revoked = true WHERE user_id = '...'`
- The user's active session is terminated the moment their current 15-minute Access Token expires.

**State lifecycle:**

```text
refresh_tokens.state:   ACTIVE ──► EXPIRED (Auto-logout triggered by time)
                           └─────► REVOKED (Manual logout / forced by admin)

access_tokens (JWT):    VALID ───► EXPIRED (Requires fresh JWT via refresh_token)




