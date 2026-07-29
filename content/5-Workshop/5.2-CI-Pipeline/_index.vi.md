---
title: "CI Pipeline (Cổng chất lượng)"
date: 2026-07-30
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

Trước khi deploy bất cứ thứ gì, mọi pull request trong repo backend phải đi qua một **cổng chất lượng CI** (`.github/workflows/ci.yml`). Bước này không tốn chi phí AWS và giữ cho branch deploy luôn ở trạng thái sẵn sàng release.

#### 1. Thiết kế pipeline

Repo chứa hai vùng (`backend/` monolith, `lambdas/` các function thanh toán), nên workflow trước hết phát hiện **path nào thay đổi** và chỉ chạy job tương ứng:

```yaml
changes:
  permissions:
    pull-requests: read   # bắt buộc cho paths-filter
    contents: read
  steps:
    - uses: actions/checkout@v5
    - id: f
      uses: dorny/paths-filter@v4
      with:
        filters: |
          backend: ["backend/**"]
          lambdas: ["lambdas/**"]
```

Job backend sau đó tái tạo điều kiện production: một **service container PostgreSQL 16** thật, schema qua **Alembic**, rồi lint + test:

```yaml
test-backend:
  services:
    postgres:
      image: postgres:16
      env: { POSTGRES_USER: app, POSTGRES_PASSWORD: app, POSTGRES_DB: courtbooking_dev }
  env:
    DATABASE_URL: postgresql+psycopg://app:app@localhost:5432/courtbooking_dev
  steps:
    - uses: actions/checkout@v5
    - uses: actions/setup-python@v5
      with: { python-version: "3.12" }
    - run: cd backend && python -m pip install -r requirements-dev.txt
    - run: cd backend && ruff check .
    - run: |
        cd backend
        alembic upgrade head   # schema + extension btree_gist
        python -m pytest       # fixture theo session reseed dữ liệu, rồi 52 test chạy
```

Hai chi tiết giúp pipeline đáng tin cậy:

- **Migrate trước khi test** — `alembic upgrade head` dựng schema (bao gồm extension `btree_gist` dùng cho exclusion constraint chống đặt trùng sân) trước khi pytest chạm vào DB.
- **Test tự seed dữ liệu** — một autouse fixture scope session trong `tests/conftest.py` nạp seed SQL một lần mỗi lượt chạy, giúp bộ test idempotent ở local *và* đồng thời là bước seed duy nhất mà CI cần.

#### 2. Các lỗi đã gặp (và bài học)

**`Resource not accessible by integration`** — `dorny/paths-filter` đọc PR qua GitHub API, nhưng `GITHUB_TOKEN` mặc định không có quyền `pull-requests: read`. Sửa bằng block `permissions:` tường minh ở trên.

**Lint pass ở local, fail trên CI với 46 lỗi** — cái bẫy *tool không pin version* kinh điển. `requirements-dev.txt` ghi `ruff` không kèm version; CI cài **ruff 0.16.0** vừa phát hành (bộ rule mặc định mở rộng thêm isort/pyupgrade/bugbear), trong khi máy local đang ở **0.15.21**:

![Runtime version conflict in CI](/images/5-Workshop/5.2/gh_ci_runtime_version_conflict.png)

Chúng tôi chứng minh bằng cách chạy cả hai version trên cùng một code — 0.15.21: `All checks passed`; 0.16.0: `46 errors`. Cách sửa: **pin linter** (`ruff==0.15.21`) để CI và local không bao giờ lệch nhau.

{{% notice tip %}}
Hãy pin mọi tool đứng ở cổng build. Một linter/formatter không pin version nghĩa là CI có thể bắt đầu fail dù **không có thay đổi code nào** — lỗi trông như của bạn nhưng thực ra đến từ phía nhà cung cấp.
{{% /notice %}}

**Test fail chập chờn (409/400 thay vì 201/200)** — ba test booking chỉ fail khi chạy lại. Nguyên nhân gốc là **state pollution**: các lượt chạy trước đã làm biến đổi dữ liệu seed (một booking bị hủy, một slot PENDING còn sót), chứ không phải vấn đề timezone như bề ngoài. Cơ chế auto-reseed trong conftest ở trên chính là cách sửa — mỗi session pytest đều bắt đầu từ một bộ dữ liệu xác định.
