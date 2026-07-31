---
title: "Architecture Diagram Best Practices"
date: 2026-07-31
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# Architecture Diagram Best Practices

A good architecture diagram is not decoration — it is how a design is communicated, reviewed, and defended. During this project our first diagrams were rejected in mentor review ("where is the VPC?", "why is GitHub inside AWS?"), and fixing them taught me that a diagram follows rules just like code does. This blog collects the practices that turned our rejected draft into a clean, review-ready diagram.

**Reference:** [How to draw professional AWS architecture diagrams (draw.io)](https://youtu.be/l8isyDe-GwY)

---

### 1. Use the official, current AWS icon set

Mixing old and new AWS icons — or using the incomplete defaults built into a tool — looks unprofessional to reviewers. Download the **official AWS Architecture Icons** and use one consistent generation throughout. In draw.io, keep every service icon at a **uniform size** (e.g. 60 px) so the diagram reads evenly.

### 2. Respect the resource hierarchy

Containers must nest in the correct order, drawn as labelled group frames:

```
AWS Cloud  →  Region  →  VPC  →  Availability Zone  →  Subnet  →  resource
```

A resource sitting in the wrong frame is a correctness error, not a style one. Reviewers read the nesting to understand *where* a service actually lives.

### 3. Put each service in its correct scope

This was the single most common mistake in our review. Services belong to different scopes, and the diagram must show it:

- **Global** (outside the Region): IAM, Route 53, CloudFront.
- **Regional** (in the Region, outside the VPC): Amplify, Cognito, S3, API Gateway, Lambda, SNS, SSM.
- **VPC / subnet-bound**: ALB and NAT (public subnets), EC2 and RDS (private subnets).

Two rules that follow from this: draw **Availability Zones so they slightly extend beyond the VPC** (they are physical infrastructure, not a VPC sub-part), and represent an **Auto Scaling Group as a frame** around its instances — never as a single icon.

### 4. Keep non-AWS actors outside the AWS Cloud boundary

Users, the developer, external payment gateways, and **GitHub** are not AWS resources — they must sit *outside* the AWS Cloud frame. Putting GitHub inside AWS was a real error a mentor flagged in our draft.

### 5. Show the network path

A grader should be able to trace one request end-to-end without guessing: user → Internet Gateway → ALB → EC2 → RDS. Show the Internet Gateway on the VPC boundary, the NAT Gateway's egress path for private subnets, and RDS replication between AZs. Number the steps and list them in a side text box.

### 6. A consistent visual language

- **Uniform icon size**, and a legend explaining every arrow and colour.
- **Colour-code borders** — e.g. one colour for main services, another for sub-features — and give arrows a consistent meaning (solid = request path, dashed = operational/CI-CD flows).
- **Protect readability**: give text boxes a solid white background so lines and arrows don't cross through labels.
- Use **"To Back" (Cmd/Ctrl+Shift+B)** to push large group frames behind the icons, so the icons stay selectable.

### 7. Reuse via custom libraries and a team repo

Save pre-formatted components (or a whole VPC layout) as a **custom draw.io library** so you don't reformat every time. Share the libraries and `.drawio` source files in a central team repository so everyone follows the same style — never copy random diagrams off the internet.

---

### Takeaway

The lesson that stuck with me: **an architecture diagram is a technical artifact with correctness rules, not a picture.** Getting the scopes, hierarchy, and network path right is what makes it survive review — and forces you to actually understand where every service in your design lives.
