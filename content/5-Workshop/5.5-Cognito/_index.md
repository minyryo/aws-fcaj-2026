---
title: "Authentication (Cognito)"
date: 2026-07-30
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

Amazon Cognito is the project's **auth authority**: it stores users, verifies passwords, and issues JWTs. A key design decision shapes everything here:

{{% notice note %}}
**The frontend never talks to Cognito.** There is no Amplify Auth SDK in the SPA. The backend *wraps* Cognito (`app/cognito.py`): the FE calls `POST /api/v1/auth/login` on our API; the BE calls Cognito, returns the tokens; the FE just stores them and sends `Authorization: Bearer <token>` on every request. Result: zero Cognito configuration in the frontend, and the auth flow can be swapped or mocked server-side.
{{% /notice %}}

#### 1. User pool, app client, and groups

Three resources, created once:

```bash
# 1. The pool — email is the username, self-verified
POOL_ID=$(aws cognito-idp create-user-pool --pool-name court-booking \
  --username-attributes email --auto-verified-attributes email \
  --query 'UserPool.Id' --output text --profile thanh)

# 2. A PUBLIC app client for the SPA — no secret (a browser cannot keep one)
CLIENT_ID=$(aws cognito-idp create-user-pool-client --user-pool-id $POOL_ID \
  --client-name court-booking-spa --no-generate-secret \
  --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH ALLOW_USER_SRP_AUTH \
  --query 'UserPoolClient.ClientId' --output text --profile thanh)

# 3. Groups = application roles
for g in player court_manager admin; do
  aws cognito-idp create-group --user-pool-id $POOL_ID --group-name "$g" --profile thanh
done
```

![Cognito setup](/images/5-Workshop/5.5/cognito_setup.png)

The three groups mirror the role model exactly: a user's group appears in their JWT as the `cognito:groups` claim; the backend caches it into `users.role`, which `require_role("admin")` / `require_manager` dependencies check on every protected endpoint. Changing a user's role in production means changing their **Cognito group first**, then the DB cache — Cognito stays the source of truth.

#### 2. Wiring it to the backend

The pool and client IDs go into SSM (section 5.4) and reach the app via the env bridge:

```bash
aws ssm put-parameter --name /court-booking/dev/COGNITO_USER_POOL_ID --type String --value "$POOL_ID"   --overwrite --profile thanh
aws ssm put-parameter --name /court-booking/dev/COGNITO_CLIENT_ID    --type String --value "$CLIENT_ID" --overwrite --profile thanh
```

The switch is automatic: while `APP_ENV=local`, `auth.py` uses a **local-mode fallback** (a Bearer token is matched directly against seeded `cognito_sub` values — this is what lets the walking-skeleton deploy in 5.6 work before Cognito exists). Once `APP_ENV=dev` and the two IDs are present, `cognito.py` calls the real pool — same endpoints, same frontend code, real JWTs.

{{% notice tip %}}
Deferred on purpose: Google OAuth (`POST /auth/oauth`) additionally requires a Cognito **hosted-UI domain + identity provider**. Email/password auth needs none of that, so social login is a later, independent step.
{{% /notice %}}
