---
title: "Event 1 — Data Driven, AI Risen"
date: 2026-06-27
weight: 1
chapter: false
pre: " <b> 4.1. </b> "
---

### Event Information

| Field | Details |
|-------|---------|
| **Event Name** | Data Driven, AI Risen |
| **Date & Time** | 09:00, June 27, 2026 |
| **Location** | Livestream |
| **Role** | Attendee |

---

### Event Description

The FCAJ Community Day in June 2026 featured five presentations focused on practical applications of Artificial Intelligence across cloud infrastructure, development, and enterprise workflows.

#### 1. Career Development and AI in Cloud Operations
*Speaker: Steve Tran (Cloud Thinker)*

Steve traced his path from IT helpdesk to AWS Solution Architect, highlighting how AI is simultaneously accelerating coding speed while raising the bar for engineers managing complex, modernized systems. He introduced Cloud Thinker's AI agentic platform, which automates incident resolution, security testing, quality control, and FinOps (cloud cost optimization).

#### 2. Building Voice AI Agents for the Vietnamese Market
*Speakers: Hieu Nghi (Renova Cloud), Kiet (Student Video Group), Trung (R AI)*

The session featured a live demo of a Voice AI agent built on AWS Bedrock and examined the challenges of deploying voice agents in the Vietnamese enterprise context. Trung proposed a custom Speech-to-Text → LLM → Text-to-Speech pipeline to accurately handle regional accents, natural conversation interruptions, and enterprise tool-calling tasks such as automatically blocking a bank card.

#### 3. Automated Troubleshooting with AWS DevOps Agent
*Speakers: Bao and Nguyen (Cloud Kinetics)*

The speakers demonstrated how the AWS DevOps Agent addresses fragmented logs and slow incident response by automatically investigating alerts, identifying root causes, and generating step-by-step mitigation plans. A live demo showed the agent diagnosing a simulated DDoS attack on an ECS-hosted e-commerce application in minutes rather than hours.

#### 4. Transforming Human Resources with Amazon Q
*Speakers: Truong and Minh Anh (Noventiq)*

Minh Anh outlined HR pain points such as manual CV screening and the risks of uploading applicant data to public AI tools, while Truong demonstrated Amazon Q as a secure, enterprise-customizable AI assistant. The live demo showed Amazon Q reading multiple candidate CVs, matching them against a Cloud Engineer job description, and producing a scored evaluation report automatically.

#### 5. Securing Amazon Q Connections with Private MCP Servers
*Speakers: Toan Nguyen and Nghi*

The session covered the security risks of exposing Model Context Protocol (MCP) servers — which connect AI assistants like Amazon Q to tools such as Jira, Gmail, or Zalo — to the public internet (DDoS, Man-in-the-Middle attacks). Toan detailed an architecture using AWS VPC connections, private subnets, interface endpoints, and Route 53 resolvers to ensure all AI queries and data traffic remain strictly within the private AWS network.

---

### Outcomes and Value Gained

- **Broader view of AI's impact on cloud careers:** The career talk reframed how I think about upskilling — AI tools raise baseline productivity but also demand deeper system-level expertise, which directly motivates investing more in AWS certification study.

- **Practical architecture insight for voice/notification features:** The Voice AI session's custom Vietnamese pipeline (STT → LLM → TTS) is relevant context for our court booking app — if we extend notifications beyond SNS to voice or chat, we now understand the non-trivial engineering involved.

- **Incident response automation as a team strategy:** Learning about the AWS DevOps Agent's ability to reduce troubleshooting from hours to minutes is directly applicable to our project's future monitoring setup; this will inform how we configure CloudWatch alarms and runbooks.

- **Secure internal AI tooling model:** The Amazon Q HR demo showed how AI assistants can be deployed within enterprise guardrails — a pattern we can reference if the team considers integrating AI into internal workflows without exposing sensitive project data.

- **Security posture for AI integrations:** Understanding the MCP server vulnerability and the private-subnet mitigation pattern deepens our team's awareness of API security, complementing the API Gateway + Lambda architecture we are already building.
