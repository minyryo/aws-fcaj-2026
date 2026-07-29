---
title: "CI Pipeline (Quality Gate)"
date: 2026-07-30
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

Before deploying anything, every pull request in the backend repo passes a **CI quality gate** (`.github/workflows/ci.yml`). This costs nothing on AWS and keeps the deploy branch always releasable.

#### 1. Pipeline design

The repo hosts two zones (`backend/` monolith, `lambdas/` payment functions), so the workflow first detects **which paths changed** and only runs the relevant job:

```yaml
changes:
  permissions:
    pull-requests: read   # required by paths-filter
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

The backend job then reproduces production conditions: a real **PostgreSQL 16 service container**, schema via **Alembic**, then lint + tests:

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
        alembic upgrade head   # schema + btree_gist extension
        python -m pytest       # session fixture reseeds, then 52 tests run
```

Two details make this reliable:

- **Migrate before test** — `alembic upgrade head` builds the schema (including the `btree_gist` extension used by the double-booking exclusion constraint) before pytest touches the DB.
- **Self-seeding tests** — a session-scoped autouse fixture in `tests/conftest.py` loads the seed SQL once per run, making the suite idempotent locally *and* the only seed step CI needs.

#### 2. Problems we hit (and what they taught)

**`Resource not accessible by integration`** — `dorny/paths-filter` reads the PR via the GitHub API, but the default `GITHUB_TOKEN` had no `pull-requests: read` permission. Fixed by the explicit `permissions:` block above.

**Lint passed locally, failed in CI with 46 errors** — the classic *unpinned tool* trap. `requirements-dev.txt` listed `ruff` with no version; CI installed the just-released **ruff 0.16.0** (whose default rule set expanded to include isort/pyupgrade/bugbear), while the local machine had **0.15.21**:

![Runtime version conflict in CI](/images/5-Workshop/5.2/gh_ci_runtime_version_conflict.png)

We proved it by running both versions on the same code — 0.15.21: `All checks passed`; 0.16.0: `46 errors`. Fix: **pin the linter** (`ruff==0.15.21`) so CI and local can never drift.

{{% notice tip %}}
Pin every tool that gates a build. An unpinned linter/formatter version means CI can start failing with **zero code changes** — the failure looks like your fault but is a supply-side change.
{{% /notice %}}

**Flaky test failures (409/400 where 201/200 expected)** — three booking tests failed only on re-runs. Root cause was **state pollution**: earlier runs had mutated seed rows (a cancelled booking, a leftover PENDING slot), not the timezone issue it resembled. The conftest auto-reseed above is the fix — every pytest session starts from a known dataset.
