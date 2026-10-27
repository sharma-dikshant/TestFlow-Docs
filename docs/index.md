---
title: TestFlow
description: Home page of the TestFlow backend documentation, with a reading guide and glossary
status: stable
last_updated: 2026-10-27
---

# TestFlow Backend Docs

**TestFlow** is an online assessment platform. People and organizations use it to create tests (often with AI), invite participants, run timed attempts, and score the answers automatically. It also builds AI learning roadmaps and offers an AI assistant that can answer questions about your tests and results.

The backend is one NestJS application (a "modular monolith"). It uses PostgreSQL for data, Redis for caching and rate limiting, RabbitMQ for background jobs, Socket.IO for realtime messages, and LangChain/LangGraph to talk to LLM providers (OpenAI, Anthropic, Google).

!!! tip "New to the team?"
    Read these four pages in order: [Overview](overview.md), [Getting Started](getting-started.md), [Architecture](architecture.md), then the [ERD](erd.md). After that, jump to the feature page you are working on.

---

## How to read these docs

### Start here

| Page | What you will learn |
| --- | --- |
| [Overview](overview.md) | What the product does, who it is for, and what is built today. |
| [Getting Started](getting-started.md) | How to run the backend on your machine: tools, env vars, migrations, seeds, tests. |
| [Architecture](architecture.md) | The module map, the request pipeline, the infrastructure, and the shared code conventions. |

### Data

| Page | What you will learn |
| --- | --- |
| [ERD (Data Model)](erd.md) | All database tables and how they relate. |
| [Data Access Patterns](data-access-pattern.md) | How the code reads and writes data: repositories, query builders, transactions, caching. |

### API

| Page | What you will learn |
| --- | --- |
| [API v1](api-design/v1.md) | Every HTTP endpoint, its auth needs, and the response envelope. |

### Feature design (HLD)

"HLD" means high-level design. Each page explains one feature: the flow, the states, and the main files.

| Page | What you will learn |
| --- | --- |
| [Authentication & Users](hld-design/auth.md) | Signup, login, Google sign-in, JWT access/refresh tokens, email verification, password reset. |
| [RBAC](hld-design/rbac.md) | Roles and permissions, for the platform and inside organizations. |
| [Organizations](hld-design/organizations.md) | Creating organizations, members, org roles, invitations, and bulk CSV invites. |
| [Assessments](hld-design/assessments.md) | The assessment lifecycle (draft, ready, published), participants, invitations, templates, and prompts. |
| [Questions & Tags](hld-design/questions-and-tags.md) | Question types (MCQ, coding, descriptive), the question bank, and tags. |
| [AI Assessment Generation](hld-design/assessment-generation.md) | How a prompt becomes a full assessment through background section jobs. |
| [Attempts](hld-design/attempts.md) | Starting, answering, submitting, and reviewing an attempt; leaderboard and analytics. |
| [Evaluation](hld-design/evaluation.md) | How submitted attempts are scored per question type, in the background. |
| [Code Execution](hld-design/code-execution.md) | How candidate code runs on Piston or OnlineCompiler.io and is judged against test cases. |
| [AI Layer](hld-design/ai.md) | The chat model factory, the model registry, LangGraph graphs, prompts, structured output, and LLM audit. |
| [AI Assistant & Tools](hld-design/ai-assistant.md) | The conversational assistant and the read-only tools it can call. |
| [Commerce, Payments & Webhooks](hld-design/commerce.md) | Plans, subscriptions, usage quotas, Razorpay checkout, and webhooks. |
| [Learning Roadmaps](hld-design/learning-roadmaps.md) | AI-built learning graphs, progress tracking, and roadmap assessments. |
| [Realtime Events](hld-design/realtime-events.md) | The Socket.IO gateway: auth, rooms, events, and live code runs. |
| [Jobs & Schedulers](hld-design/jobs-and-schedulers.md) | The `jobs` table, job status, retries, and cron tasks. |
| [Notifications](hld-design/notifications.md) | Transactional email over SMTP and the email templates. |
| [Platform Services](hld-design/platform-services.md) | Rate limiting, feature flags, health checks, stats, newsletters, support tickets, and file storage. |

---

## Glossary

| Term | Meaning |
| --- | --- |
| **Assessment** | A test. It has a title, a time window, a duration, a marking scheme, sections, and questions. Its status moves `draft` → `ready` → `published`. |
| **Section** | A part of an assessment that holds questions of one type (for example, 10 MCQs). AI generation creates one background job per section. |
| **Question** | One item to answer. The types are `mcq` (multiple choice), `coding` (write code that passes test cases), and `descriptive` (free text, graded by an LLM). |
| **Question bank** | The shared `questions` table. Questions are copied ("snapshotted") into an assessment, so later edits do not change old tests. |
| **Participant** | A user linked to an assessment. Status is `invited`, `active`, or `revoked`. The creator is also a participant but does not use a seat. |
| **Invitation** | A request to join something. Assessment invitations add participants. Organization invitations add members, one by one or from a CSV file. |
| **Attempt** | One sitting of an assessment by a participant. Status moves `in_progress` → `submitted` → `evaluating` → `evaluated`. A reviewer can mark it `rejected`. |
| **Response** | A participant's answer to one question inside an attempt. |
| **Evaluation** | Scoring a submitted attempt. It runs in the background and is split by question type. |
| **Template** | A ready-made starting point. *Assessment templates* define sections and question counts. *Roadmap templates* give a starting point for a learning roadmap. |
| **Assessment prompt** | A saved, reusable prompt for AI generation. It can be public or private and can carry tags. |
| **Tag** | A label (for example, "javascript") used to group and search content. |
| **Learning roadmap** | An AI-generated graph of topics (nodes and edges) that a learner works through, with progress tracking and practice assessments per node. |
| **Organization (org)** | A team account. Members have an org role: `org_owner`, `org_admin`, or `org_member`. Many routes can act "inside an org" through the `x-organization-id` header or an `:orgUid` path segment. |
| **Platform role** | A role that applies across the whole platform, such as `super_admin`. It is separate from org roles. |
| **Permission** | A named right such as `questions:create`. Roles hold permissions. Routes ask for them with `@Permissions()` or `@OrgPermissions()`. |
| **Plan** | A product tier. Each plan has versions, and each version lists features with limits (for example, how many AI assessments you can generate). |
| **Subscription** | A user's link to a plan version. Status is `PENDING`, `ACTIVE`, `PAST_DUE`, or `CANCELLED`. |
| **Usage / quota** | How much of a plan feature a user has used. AI generation spends quota and gives it back if the generation fails. |
| **Order / Payment** | An order is what the user is buying. A payment is the Razorpay charge for it. Razorpay webhooks confirm payments. |
| **Job** | A row in the `jobs` table that tracks one background task (generate a section, evaluate an attempt, import invitations). Clients poll `GET /v1/jobs/:id` to see progress. |
| **Consumer** | A class that listens to a RabbitMQ queue and runs the jobs that arrive there. |
| **AI model** | A row in the `ai_models` table. It names a provider and a model, plus the env vars that hold its API key and base URL. The user's plan decides which model they get. |
| **LLM audit** | A row in `llm_audit` that records one LLM call (model, tokens, timing) for cost tracking and debugging. |
| **AI assistant / conversation** | A chat with an LLM that can call tools to read your assessments, attempts, and roadmaps. Messages travel over the WebSocket. |
| **Feature flag** | A switch that turns a feature on for some roles (for example, `beta_tester`). |
| **Envelope** | The standard JSON shape of every response: `{ success, message, data, meta? }`. |
