---
title: "Frontend (Amplify Hosting)"
date: 2026-07-30
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

SPA React + Vite được host trên **AWS Amplify Hosting**, và chính nó *là* CI/CD của frontend: kết nối repo một lần, mỗi lần push sẽ tự động build và phát hành qua CDN toàn cầu của Amplify, kèm sẵn HTTPS. Frontend không cần bất kỳ workflow GitHub Actions nào.

#### 1. Kết nối repository

Từ Amplify console, tạo app và cấp quyền kết nối GitHub:

![Amplify start](/images/5-Workshop/5.7/amplify_init_screen.png)

![Connect GitHub](/images/5-Workshop/5.7/amplify_connect_with_github.png)

![Authorize GitHub](/images/5-Workshop/5.7/amplify_connect_with_github_1.png)

Chọn repo frontend và branch để deploy — Amplify build lại ở mỗi lần push vào branch đó:

![Select repo and branch](/images/5-Workshop/5.7/amplify_add_repo_and_branch.png)

Amplify tự nhận diện build Vite (`npm run build`, output `dist/`); xem lại app settings và xác nhận:

![App settings](/images/5-Workshop/5.7/amplify_app_settings.png)

![Review and deploy](/images/5-Workshop/5.7/amplify_review.png)

#### 2. Biến môi trường lúc build

SPA được cấu hình hoàn toàn qua **biến môi trường của Amplify** (Vite "nướng" vào bundle lúc build):

![Environment variables](/images/5-Workshop/5.7/amplify_env_var.png)

| Biến | Giá trị | Ý nghĩa |
| --- | --- | --- |
| `VITE_USE_MOCK_API` | `false` | ngừng dùng mock localStorage, gọi backend thật |
| `VITE_API_BASE_URL` | `/api` | nơi tầng service gửi request |

#### 3. Nối SPA với backend

Một trang HTTPS không thể gọi ALB chạy HTTP thuần (bị chặn mixed-content), và API thì chưa có domain riêng. Giải pháp là **reverse-proxy rewrite** của Amplify — trình duyệt luôn same-origin, Amplify chuyển tiếp `/api/*` phía server tới ALB:

```json
[
  { "source": "/api/<*>", "target": "https://<alb-dns-name>/api/<*>", "status": "200" },
  { "source": "/<*>",     "target": "/index.html",                    "status": "200" }
]
```

- Rule 1 (**200 = proxy**, không phải redirect): trình duyệt → Amplify (HTTPS) → ALB. **Không cần cấu hình CORS, không cần chứng chỉ ACM trên ALB** — trình duyệt không bao giờ thấy origin của ALB.
- Rule 2 là **SPA fallback** tiêu chuẩn: các route phía client như `/manager/courts` phải trả về `index.html` khi refresh thay vì 404.

{{% notice note %}}
Khi có custom domain (`api.<domain>` + chứng chỉ ACM trên ALB — phase G đã hoãn), rule proxy có thể bỏ: đặt `VITE_API_BASE_URL=https://api.<domain>` và thêm URL Amplify vào parameter `CORS_ORIGINS` của backend (5.4). Rewrite là đường nhanh, không phải kiến trúc cuối cùng.
{{% /notice %}}

{{% notice tip %}}
*Region* "nhà" của Amplify không ảnh hưởng người dùng cuối — nội dung luôn được phục vụ từ mạng edge CloudFront. Thứ quyết định độ trễ là vị trí của **API**, vì các request `/api/*` kết thúc ở region của ALB.
{{% /notice %}}
