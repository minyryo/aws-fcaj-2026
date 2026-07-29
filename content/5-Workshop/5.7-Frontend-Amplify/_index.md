---
title: "Frontend (Amplify Hosting)"
date: 2026-07-30
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

The React + Vite SPA is hosted on **AWS Amplify Hosting**, which *is* the frontend CI/CD: connect the repo once, and every push builds and publishes automatically over Amplify's global CDN with HTTPS included. No GitHub Actions workflow exists for the frontend.

#### 1. Connecting the repository

From the Amplify console, create an app and authorize the GitHub connection:

![Amplify start](/images/5-Workshop/5.7/amplify_init_screen.png)

![Connect GitHub](/images/5-Workshop/5.7/amplify_connect_with_github.png)

![Authorize GitHub](/images/5-Workshop/5.7/amplify_connect_with_github_1.png)

Select the frontend repository and the branch to deploy from — Amplify rebuilds on every push to it:

![Select repo and branch](/images/5-Workshop/5.7/amplify_add_repo_and_branch.png)

Amplify auto-detects the Vite build (`npm run build`, output `dist/`); review the app settings and confirm:

![App settings](/images/5-Workshop/5.7/amplify_app_settings.png)

![Review and deploy](/images/5-Workshop/5.7/amplify_review.png)

#### 2. Build-time environment variables

The SPA is configured entirely through **Amplify environment variables** (baked in at build time by Vite):

![Environment variables](/images/5-Workshop/5.7/amplify_env_var.png)

| Variable | Value | Meaning |
| --- | --- | --- |
| `VITE_USE_MOCK_API` | `false` | stop using localStorage mocks, call the real backend |
| `VITE_API_BASE_URL` | `/api` | where the service layer sends requests |

#### 3. Connecting the SPA to the backend

An HTTPS page cannot call a plain-HTTP ALB (mixed-content block), and the API has no domain of its own yet. The solution is Amplify's **reverse-proxy rewrite** — the browser stays same-origin, Amplify forwards `/api/*` server-side to the ALB:

```json
[
  { "source": "/api/<*>", "target": "https://<alb-dns-name>/api/<*>", "status": "200" },
  { "source": "/<*>",     "target": "/index.html",                    "status": "200" }
]
```

- Rule 1 (**200 = proxy**, not a redirect): browser → Amplify (HTTPS) → ALB. **No CORS configuration and no ACM certificate on the ALB are needed** — the browser never sees the ALB origin.
- Rule 2 is the standard **SPA fallback**: client-side routes like `/manager/courts` must serve `index.html` on refresh instead of 404.

{{% notice note %}}
When the custom domain lands (`api.<domain>` + ACM cert on the ALB — the deferred phase G), the proxy rule can be dropped: set `VITE_API_BASE_URL=https://api.<domain>` and add the Amplify URL to the backend's `CORS_ORIGINS` parameter (5.4). The rewrite is the fast path, not the final architecture.
{{% /notice %}}

{{% notice tip %}}
Amplify's home *region* does not affect end users — content is served from the CloudFront edge network regardless. What matters for latency is where the **API** lives, since `/api/*` requests terminate at the ALB's region.
{{% /notice %}}
