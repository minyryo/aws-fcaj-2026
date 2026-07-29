---
title: "Network & Database (RDS)"
date: 2026-07-30
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

The compute and data layer share one VPC. We deliberately **kept the default VPC** (no NAT Gateway, no custom subnets) to stay inside the Free Tier — isolation is enforced by a **security-group chain** instead.

#### 1. The security-group chain

Three SGs, each one only reachable from the previous link:

```
internet ──80/443──► court-booking-alb ──8000──► court-booking-app ──5432──► court-booking-rds
                      (ALB)                        (EC2 / FastAPI)              (RDS, not public)
```

- `court-booking-alb` — inbound 80/443 from `0.0.0.0/0`
- `court-booking-app` — inbound 8000 **only from the ALB SG** (source = security group, not CIDR)
- `court-booking-rds` — inbound 5432 **only from the app SG**

![Security group settings](/images/5-Workshop/5.3/vpc_sg_settings.png)

Adding the chained inbound rules (note the *source* is another security group):

![SG add rules](/images/5-Workshop/5.3/vpc_sg_add_rules.png)

![SG add rules — app tier](/images/5-Workshop/5.3/vpc_sg_add_rules_1.png)

![SG add rules — db tier](/images/5-Workshop/5.3/vpc_sg_add_rules_2.png)

{{% notice note %}}
Referencing a **security group as the source** (instead of an IP range) means the rule follows the instances: any EC2 launched by the Auto Scaling Group with `court-booking-app` attached can reach RDS — no IP bookkeeping.
{{% /notice %}}

#### 2. DB subnet group

RDS requires a subnet group spanning **at least two Availability Zones** (a hard requirement, even for a single-AZ instance). We grouped the default VPC's subnets:

![Create DB subnet group](/images/5-Workshop/5.3/vpc_create_subnet_group.png)

#### 3. The RDS instance

PostgreSQL 16 on `db.t4g.micro` (Free-Tier-eligible; Graviton is fine for RDS because the engine is managed — our code never runs on the DB host), 20 GB, single-AZ, and **not publicly accessible**:

![Create RDS instance](/images/5-Workshop/5.3/rds_create_instance_0.png)

![Create RDS instance — settings](/images/5-Workshop/5.3/rds_create_instance_1.png)

```bash
aws rds create-db-instance \
  --db-instance-identifier court-booking-db-dev \
  --engine postgres --engine-version 16 --db-instance-class db.t4g.micro \
  --allocated-storage 20 --db-name courtbooking \
  --master-username app --master-user-password "$DB_PASSWORD" \
  --db-subnet-group-name court-booking-db --vpc-security-group-ids $RDS_SG \
  --no-publicly-accessible --backup-retention-period 1 \
  --profile thanh
```

#### 4. Errors we hit

{{% notice warning %}}
**`FreeTierRestrictionError` — backup retention.** Our first attempt used `--backup-retention-period 7`; the account is on the new AWS **Free Tier plan**, which actively blocks settings that could bill beyond free limits. Lowering to `1` (automated backups still on, minimal footprint) passed. The same plan also blocks Multi-AZ, larger instance classes and provisioned IOPS — stay within `t4g.micro` / 20 GB / single-AZ.
{{% /notice %}}

{{% notice warning %}}
**`InvalidParameterValue: MasterUserPassword`.** RDS forbids exactly four characters in the master password: `/`, `@`, `"` and space — and ours contained `@`. Since the password is also embedded in a connection URI (`postgresql+psycopg://app:PASSWORD@host:5432/db`), we regenerated it from the **URI-safe alphabet** (`A–Z a–z 0–9 - _ . ~`), which satisfies RDS *and* needs no percent-encoding: `LC_ALL=C tr -dc 'A-Za-z0-9-_.~' < /dev/urandom | head -c 24`.
{{% /notice %}}

The schema itself (7 tables, the `btree_gist` extension, the anti-double-booking exclusion constraint) is **not** created here — it arrives via Alembic migrations during the first deploy (section 5.6), because the DB is unreachable from outside the VPC by design.
