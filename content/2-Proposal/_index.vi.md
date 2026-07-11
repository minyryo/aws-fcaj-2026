---
title: "Bản đề xuất"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

Bản đề xuất cho **Ứng dụng Đặt sân thể thao** — kiến trúc AWS hybrid chia hệ thống thành EC2 monolith (xác thực + đặt sân) và luồng thanh toán serverless — được tổ chức thành hai phần:

**2.1** [Thiết kế kiến trúc](2.1-architecture/) — tóm tắt điều hành, tuyên bố vấn đề, kiến trúc giải pháp hybrid với diễn giải luồng 14 bước, thiết kế API & cơ sở dữ liệu thống nhất, lộ trình, ước tính ngân sách, đánh giá rủi ro và kết quả kỳ vọng.

**2.2** [Quy trình triển khai (CI/CD)](2.2-deployment/) — thiết kế CI/CD (GitHub Actions điều phối Amplify, CodeDeploy và SAM) cùng tài liệu hướng dẫn thiết lập từng bước cho nhóm, bao gồm quality gate cho PR, migration cơ sở dữ liệu và quy trình rollback.
