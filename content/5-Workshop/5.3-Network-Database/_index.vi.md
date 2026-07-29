---
title: "Network & Database (RDS)"
date: 2026-07-30
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

Tầng compute và tầng dữ liệu dùng chung một VPC. Chúng tôi chủ động **giữ default VPC** (không NAT Gateway, không subnet tùy chỉnh) để nằm trong Free Tier — việc cô lập được thực thi bằng một **chuỗi security group** thay thế.

#### 1. Chuỗi security group

Ba SG, mỗi SG chỉ có thể được truy cập từ mắt xích liền trước:

```
internet ──80/443──► court-booking-alb ──8000──► court-booking-app ──5432──► court-booking-rds
                      (ALB)                        (EC2 / FastAPI)              (RDS, không public)
```

- `court-booking-alb` — inbound 80/443 từ `0.0.0.0/0`
- `court-booking-app` — inbound 8000 **chỉ từ SG của ALB** (source = security group, không phải CIDR)
- `court-booking-rds` — inbound 5432 **chỉ từ SG của app**

![Security group settings](/images/5-Workshop/5.3/vpc_sg_settings.png)

Thêm các inbound rule dạng chuỗi (chú ý *source* là một security group khác):

![SG add rules](/images/5-Workshop/5.3/vpc_sg_add_rules.png)

![SG add rules — tầng app](/images/5-Workshop/5.3/vpc_sg_add_rules_1.png)

![SG add rules — tầng db](/images/5-Workshop/5.3/vpc_sg_add_rules_2.png)

{{% notice note %}}
Tham chiếu **security group làm source** (thay vì dải IP) nghĩa là rule "đi theo" instance: bất kỳ EC2 nào do Auto Scaling Group khởi chạy có gắn `court-booking-app` đều kết nối được RDS — không phải quản lý IP thủ công.
{{% /notice %}}

#### 2. DB subnet group

RDS yêu cầu subnet group trải trên **ít nhất hai Availability Zone** (yêu cầu cứng, kể cả với instance single-AZ). Chúng tôi gom các subnet của default VPC:

![Create DB subnet group](/images/5-Workshop/5.3/vpc_create_subnet_group.png)

#### 3. RDS instance

PostgreSQL 16 trên `db.t4g.micro` (đủ điều kiện Free Tier; Graviton không thành vấn đề với RDS vì engine được AWS quản lý — code của chúng ta không bao giờ chạy trên host DB), 20 GB, single-AZ, và **không public**:

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

#### 4. Lỗi đã gặp

{{% notice warning %}}
**`FreeTierRestrictionError` — backup retention.** Lần đầu chúng tôi dùng `--backup-retention-period 7`; account đang ở **Free Tier plan** mới của AWS, vốn chủ động chặn các thiết lập có thể phát sinh phí vượt mức miễn phí. Hạ xuống `1` (vẫn bật automated backup, dấu chân tối thiểu) thì qua. Plan này cũng chặn Multi-AZ, instance class lớn hơn và provisioned IOPS — hãy ở trong giới hạn `t4g.micro` / 20 GB / single-AZ.
{{% /notice %}}

{{% notice warning %}}
**`InvalidParameterValue: MasterUserPassword`.** RDS cấm đúng bốn ký tự trong master password: `/`, `@`, `"` và dấu cách — và password của chúng tôi chứa `@`. Vì password còn được nhúng trong connection URI (`postgresql+psycopg://app:PASSWORD@host:5432/db`), chúng tôi tạo lại từ **bảng ký tự an toàn cho URI** (`A–Z a–z 0–9 - _ . ~`), vừa thỏa RDS *vừa* không cần percent-encoding: `LC_ALL=C tr -dc 'A-Za-z0-9-_.~' < /dev/urandom | head -c 24`.
{{% /notice %}}

Bản thân schema (7 bảng, extension `btree_gist`, exclusion constraint chống đặt trùng sân) **không** được tạo ở bước này — nó đến từ Alembic migration trong lần deploy đầu tiên (mục 5.6), vì theo thiết kế DB không thể truy cập từ ngoài VPC.
