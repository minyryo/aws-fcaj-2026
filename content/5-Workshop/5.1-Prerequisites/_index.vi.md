---
title: "Chuẩn bị & Danh tính"
date: 2026-07-30
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

Trước khi tạo bất kỳ resource nào, chúng tôi thiết lập **ai được phép làm gì**. Có ba danh tính, và mọi thao tác về sau trong workshop đều chạy dưới một trong ba:

| Danh tính | Loại | Dùng cho |
| --- | --- | --- |
| `hieu-cli` (+ MFA) | IAM user, account của Hiếu | credential *con người* để setup |
| `court-booking-admin` | IAM role, workload account | đích AssumeRole liên account cho mọi việc dựng hạ tầng |
| `gh-deploy-backend` / `gh-deploy-lambdas` | IAM role, workload account | CI deploy *không cần key* qua GitHub OIDC |

Policy chi tiết nằm trong guide của repo (`cicd-setup-guide.md` §1.0–§1.3); trang này trình bày kết quả và các lỗi đã gặp.

#### 1. Danh tính con người: AssumeRole liên account với MFA

Workload account là của Thanh, nhưng hạ tầng được quản trị từ IAM user của Hiếu. Thay vì chia sẻ access key, account của Thanh mở một role `court-booking-admin` chỉ trust user của Hiếu **khi có MFA**:

![Switch role setup](/images/5-Workshop/5.1/iam-switch_role-setup.png)

Trên console thao tác này là **Switch role**, còn trên CLI là một profile nối `source_profile` + `role_arn` + `mfa_serial`:

```ini
# ~/.aws/config
[profile thanh]
role_arn       = arn:aws:iam::<WORKLOAD_ACCOUNT>:role/court-booking-admin
source_profile = hieu
mfa_serial     = arn:aws:iam::<HIEU_ACCOUNT>:mfa/hieu-cli-ip16pm
```

![Switch role demo](/images/5-Workshop/5.1/iam-switch_role-demo.png)

Kiểm chứng — `get-caller-identity` phải hiển thị **role đã assume**, không phải IAM user:

![Verify AssumeRole](/images/5-Workshop/5.1/iam-verify_assumeRole.png)

{{% notice warning %}}
**Lỗi đã gặp: sai tên MFA device.** CLI liên tục từ chối mã MFA vì `mfa_serial` trong `~/.aws/config` ghi `.../mfa/hieu-cli` trong khi device thực tế đăng ký là `.../mfa/hieu-cli-ip16pm`. MFA serial là một ARN chính xác từng ký tự — hãy kiểm tra identifier trong IAM console và copy nguyên văn.
{{% /notice %}}

![MFA wrong device name](/images/5-Workshop/5.1/error-iam-mfa-wrong_device_name.png)

Cách sửa — xem identifier thật của device, sau đó cập nhật config:

![Verify MFA identifier](/images/5-Workshop/5.1/error-iam-mfa-wrong_device_name-solution-verify_identifier.png)

![Update aws config](/images/5-Workshop/5.1/error-iam-mfa-wrong_device_name-solution-update_aws_config.png)

#### 2. Quyền của role admin

Policy đầu tiên quá hẹp, dẫn tới chuỗi lỗi `AccessDenied` liên tiếp ("whack-a-mole" quyền) — mỗi service AWS mới lại cần thêm một statement:

![Not authorized error](/images/5-Workshop/5.1/error-iam-not_authorized.png)

Với một role *setup* (không phải role runtime), chúng tôi chốt quyền admin rộng — danh tính này tồn tại để dựng hạ tầng, được chặn bằng MFA, và không pipeline nào sử dụng nó:

![Admin role policy](/images/5-Workshop/5.1/iam-policy_court-booking-admin.png)

#### 3. Danh tính CI: GitHub OIDC (không có AWS key trong GitHub)

Pipeline deploy không bao giờ giữ access key. GitHub Actions đổi **OIDC token** lấy credential ngắn hạn bằng cách assume một role có trust policy khóa chặt đúng repo và branch:

![gh-deploy-backend role](/images/5-Workshop/5.1/iam_role_gh-deploy-backend.png)

![gh-deploy-lambdas role](/images/5-Workshop/5.1/iam_role_gh-deploy-lambdas.png)

Role backend chỉ được: upload lên deploy bucket, kích hoạt CodeDeploy, và chạy lệnh migration qua SSM. Role lambda giới hạn trong SAM/CloudFormation. Least privilege ở đây là thật — các role này chạy tự động ở mỗi lần push.

{{% notice note %}}
**Vệ sinh secret**, áp dụng xuyên suốt dự án: không có access key trong bất kỳ repo nào (OIDC loại bỏ nhu cầu đó), secret runtime chỉ nằm trong SSM Parameter Store (mục 5.4), và file `infra/.env` cục bộ chứa DB password đã được gitignore.
{{% /notice %}}
