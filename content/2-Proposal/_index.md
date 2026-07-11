---
title: "Proposal"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

The proposal for the **Court Booking Application** — a hybrid AWS architecture splitting the system into an EC2 monolith (auth + booking) and a serverless payment path — is organized in two parts:

**2.1** [Architecture Design](2.1-architecture/) — executive summary, problem statement, hybrid solution architecture with the 14-step flow walkthrough, unified API & database design, timeline, budget estimation, risk assessment, and expected outcomes.

**2.2** [Deployment Process (CI/CD)](2.2-deployment/) — the CI/CD design (GitHub Actions orchestrating Amplify, CodeDeploy, and SAM) and the complete step-by-step setup guide for the team, including PR quality gates, database migrations, and the rollback runbook.
