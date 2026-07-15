---
title: Entity Relationship Diagram (ERD)
description: Current-state ERD for the TestFlow database schema, verified against entity files and the full migration history (22 migrations, 2026-06-10 → 2026-07-13)
status: stable
last_updated: 2026-07-15
---

# Entity Relationship Diagram (ERD)

This page reflects the schema as implemented today — verified directly against TypeORM entity files (not migration files alone, since one migration already superseded another — see [Payments / Orders](#4-payments--orders) below). 20 entities across 4 domains: Identity & RBAC, Assessment Engine, Commerce, and Payments.

All tables use `timestamptz` for timestamps and soft-delete (`deleted_at`) unless noted otherwise. Junction/snapshot tables (`assessments_questions`, `assessment_attempt_responses`, `webhook_events`) do not soft-delete.

## 1. Identity & RBAC

```mermaid
erDiagram

    users {
        int id PK
        varchar full_name "200"
        varchar first_name "100"
        varchar last_name "100"
        varchar username "UNIQUE, 100"
        varchar email "UNIQUE, 255"
        varchar password "nullable — reserved for OAuth-only accounts, no OAuth flow implemented yet"
        varchar phone_number "UNIQUE, nullable, 20"
        boolean is_active "default true"
        boolean is_verified "default false"
        varchar oauth_provider "ENUM: google, github — nullable; column exists, no OAuth flow implemented"
        varchar oauth_id "nullable"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    refresh_tokens {
        int id PK
        varchar token_hash "SHA-256 hex of raw JWT refresh token"
        int user_id FK "FK -> users.id, CASCADE"
        timestamp expires_at
        timestamp created_at
        timestamp deleted_at "soft delete on rotation/logout"
    }

    roles {
        int id PK
        varchar key "UNIQUE, 100 — e.g. super_admin"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    resources {
        int id PK
        varchar key "UNIQUE, 100 — e.g. users:read, questions:create, *:* wildcard"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    role_permissions {
        int id PK
        int role_id FK "FK -> roles.id, CASCADE"
        int resource_id FK "FK -> resources.id, CASCADE"
        text description "nullable"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    users_roles {
        int id PK
        int user_id FK "FK -> users.id, CASCADE"
        int role_id FK "FK -> roles.id, CASCADE"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    users ||--o{ refresh_tokens : issues
    users ||--o{ users_roles : has
    roles ||--o{ users_roles : assigned_to
    roles ||--o{ role_permissions : grants
    resources ||--o{ role_permissions : granted_via
```

**Notes:**

- Composite partial unique index on `users(oauth_provider, oauth_id)` `WHERE oauth_id IS NOT NULL` — enforced at the DB level even though no OAuth flow currently populates these columns.
- `refresh_tokens` is a single-active-token-per-user model: `AuthService.storeRefreshToken()` upserts by `user_id` (resurrecting a soft-deleted row if present) rather than keeping one row per device/session.
- `resources.key` values are composite `resource:action` strings (`users:read`, `questions:create`, …) plus one wildcard `*:*` — seeded only for the `super_admin` role, which is the only role seeded out of the box.
- `role_permissions` and `users_roles` are plain join tables with their own soft-delete/audit columns, not TypeORM `@ManyToMany` relations.

## 2. Assessment Engine

```mermaid
erDiagram

    questions {
        int id PK
        text question_text
        jsonb options "array of {id, text}"
        int correct_option_id
        int created_by FK "FK -> users.id, CASCADE"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    assessments {
        int id PK
        text description
        timestamp start_time
        timestamp end_time
        int duration "minutes, max 180"
        int max_participants
        int current_participants "default 0"
        int max_total_questions
        int curr_total_questions "default 0"
        decimal correct_marks "5,2"
        decimal incorrect_marks "5,2"
        int created_by FK "FK -> users.id, CASCADE"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    assessments_questions {
        int id PK
        int assessment_id FK "FK -> assessments.id, CASCADE"
        int question_id FK "FK -> questions.id, CASCADE"
        text question_text "snapshot at assignment time"
        jsonb options "snapshot"
        int correct_option_id "snapshot"
        decimal correct_marks "5,2 — per-question override"
        decimal incorrect_marks "5,2 — per-question override"
        int order_index
        timestamp created_at
    }

    assessments_participants {
        int id PK
        int user_id FK "FK -> users.id, CASCADE"
        int assessment_id FK "FK -> assessments.id, CASCADE"
        varchar status "ENUM: not_started, in_progress (unused), completed"
        timestamp completed_at "nullable"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    assessment_attempts {
        int id PK
        uuid uid "generated, public attempt identifier used in all routes"
        int assessment_id FK "FK -> assessments.id, CASCADE"
        int user_id FK "FK -> users.id, CASCADE"
        timestamp started_at
        timestamp submitted_at "nullable"
        decimal score "8,2, nullable"
        int total_questions "snapshot of assessment.curr_total_questions at start"
        int attempted_questions "default 0"
        int correct_responses "default 0"
        int incorrect_responses "default 0"
        varchar status "ENUM: IN_PROGRESS, SUBMITTED, EVALUATED — default IN_PROGRESS"
        timestamp created_at
        timestamp updated_at
    }

    assessment_attempt_responses {
        int id PK
        int attempt_id FK "FK -> assessment_attempts.id, CASCADE"
        int assessment_question_id FK "FK -> assessments_questions.id, CASCADE"
        int selected_option_id
        boolean is_correct "computed at answer time"
        decimal marks_awarded "8,2 — computed at answer time, can be negative"
        timestamp answered_at "updated on every upsert"
    }

    users ||--o{ questions : creates
    users ||--o{ assessments : creates
    assessments ||--o{ assessments_questions : includes
    questions ||--o{ assessments_questions : mapped_in
    users ||--o{ assessments_participants : participates
    assessments ||--o{ assessments_participants : has
    users ||--o{ assessment_attempts : makes
    assessments ||--o{ assessment_attempts : attempted
    assessment_attempts ||--o{ assessment_attempt_responses : contains
    assessments_questions ||--o{ assessment_attempt_responses : answered_for
```

**Notes:**

- `assessments_questions` and `assessment_attempt_responses` are the only two tables without `deleted_at` — questions and responses are snapshot/append-only records, not independently soft-deletable.
- Unique index `(assessment_id, question_id)` on `assessments_questions` — a question can only be added to the same assessment once.
- Unique constraint `(user_id, assessment_id)` on `assessment_attempts` — enforces one attempt per user per assessment at the DB level (added in a later migration, `20260618000004`).
- No unique constraint exists on `assessment_attempt_responses(attempt_id, assessment_question_id)` — the one-response-per-question invariant is enforced only in `AttemptsService.upsertResponse()` application logic (find-then-update-or-create), not the schema.
- `assessments_participants.status = in_progress` is defined in the enum but never actually set by any code path — participants transition directly `not_started → completed` via `markParticipantCompleted()`.

## 3. Commerce — Plans, Subscriptions & Usage

```mermaid
erDiagram

    plans {
        int id PK
        varchar name "ENUM: FREE, PRO"
        varchar code "UNIQUE, 50 — generated 10-char code, e.g. 7K9XQPLM2R"
        text description
        boolean is_active "default true"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    plan_versions {
        int id PK
        int plan_id FK "FK -> plans.id, CASCADE"
        int version
        decimal price "10,2"
        varchar currency "3"
        varchar billing_cycle "ENUM: MONTHLY, YEARLY"
        boolean is_current "default true"
        timestamp effective_from
        timestamp effective_until "nullable"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    plan_features {
        int id PK
        int plan_version_id FK "FK -> plan_versions.id, CASCADE"
        varchar feature_name "100"
        text description "nullable"
        varchar feature_value "255"
        int reset_period "nullable, days — null = never resets"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    subscriptions {
        int id PK
        int user_id FK "FK -> users.id, CASCADE"
        int plan_version_id FK "FK -> plan_versions.id, CASCADE — no direct FK to plans"
        varchar status "ENUM: PENDING, ACTIVE, PAST_DUE, CANCELLED, EXPIRED"
        timestamp starts_at
        timestamp expires_at
        timestamp cancelled_at "nullable"
        boolean auto_renew "default true"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    user_feature_usage {
        int id PK
        int user_id FK "FK -> users.id, CASCADE"
        int plan_feature_id FK "FK -> plan_features.id, CASCADE"
        int used "default 0"
        date period_start
        date period_end
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    users ||--o{ subscriptions : purchases
    users ||--o{ user_feature_usage : consumes
    plans ||--o{ plan_versions : has_versions
    plan_versions ||--o{ plan_features : defines
    plan_versions ||--o{ subscriptions : "purchased as"
    plan_features ||--o{ user_feature_usage : tracked_by
```

**Notes:**

- Table name is `plan_features` — the TypeORM entity class is named `PlanVersionFeature`, which can read as if the table were `plan_version_features`. It is not; the class name and table name intentionally diverge.
- Partial unique index `(plan_id) WHERE is_current = true` on `plan_versions` — DB-enforced guarantee that only one version per plan can be current. `PlansService.createVersion()` demotes the previous current version (setting its `effective_until`) inside the same transaction before inserting the new one.
- `subscriptions` links to `plan_versions`, **not** directly to `plans` — the plan is reached transitively (`subscription → plan_version → plan`).
- `UsageService` currently has no methods implemented — the `user_feature_usage` table and `plan_features.reset_period` column exist and are schema-ready, but nothing in the codebase writes to or reads from this table yet.
- No code path currently transitions a subscription to `PAST_DUE`, `CANCELLED`, or `EXPIRED` — those enum values are defined but unused; only `PENDING → ACTIVE` (via webhook-driven payment capture) exists today.

## 4. Payments & Orders

The schema evolved mid-project: payments originally pointed directly at subscriptions; a generic `orders` abstraction was introduced later and payments were repointed to it (migration `20260713000001-change-payments-subscription-to-order.ts`). The diagram below reflects the **current** (post-migration) state.

```mermaid
erDiagram

    orders {
        int id PK
        int user_id FK "FK -> users.id, CASCADE"
        varchar resource_type "ENUM: SUBSCRIPTION, AI_CREDITS"
        int resource_id "polymorphic — no DB FK; points at subscriptions.id when resource_type=SUBSCRIPTION"
        decimal total_amount "10,2"
        varchar currency "3"
        varchar provider "ENUM: STRIPE, RAZORPAY — only RAZORPAY implemented"
        varchar provider_order_id "UNIQUE"
        varchar status "ENUM: PENDING, PAID, FAILED, FULFILLED"
        jsonb metadata "nullable"
        timestamp expires_at "payment window expiry"
        timestamp created_at
        timestamp updated_at
    }

    payments {
        int id PK
        int order_id FK "FK -> orders.id, CASCADE — NOT subscription_id (dropped by migration 20260713000001)"
        decimal amount "10,2"
        varchar currency "3"
        varchar provider "ENUM: STRIPE, RAZORPAY"
        varchar provider_order_id "nullable, partial UNIQUE where not null"
        varchar provider_payment_id "nullable, partial UNIQUE where not null"
        varchar status "ENUM: PENDING, SUCCESS, FAILED, REFUNDED"
        text failure_reason "nullable"
        timestamp paid_at "nullable"
        timestamp created_at
        timestamp updated_at
    }

    webhook_events {
        int id PK
        varchar provider "50 — plain string, e.g. RAZORPAY, not an enum type"
        varchar event_id "UNIQUE, 255 — composite key: '{event}:{razorpayPaymentId}'"
        varchar event_type "100 — e.g. payment.captured, payment.failed"
        jsonb payload "full raw webhook body"
        boolean processed "default false"
        timestamp processed_at "nullable"
        timestamp created_at
    }

    users ||--o{ orders : places
    orders ||--o{ payments : "paid via"
    subscriptions ||--o| orders : "fulfilled by (application-level, no DB FK)"
```

**Notes:**

- `orders.resource_id` is a plain int with no database-level foreign key — the link to `subscriptions.id` (or a future `ai_credits` resource) is an application-level convention, backed only by a non-unique composite index on `(resource_type, resource_id)`.
- `payments` has neither `deleted_at` nor a `subscription_id` column — payments are immutable audit records tied to exactly one order.
- `webhook_events.event_id` is **not** the raw Razorpay webhook ID; it's synthesized as `"{event}:{razorpayPaymentId}"` (e.g. `payment.captured:pay_ABC123`) and used as the idempotency key so a redelivered webhook is a no-op.
- Only `PaymentProvider.RAZORPAY` has an actual SDK integration (`razorpay` npm package + `RazorpayService`). `STRIPE` exists as an enum value on both `orders` and `payments` but has no corresponding service or dependency.
- `OrderResourceType.AI_CREDITS` is defined but has no fulfillment handler — `PaymentsService.markPaymentCaptured()` only acts on `resource_type === SUBSCRIPTION`, with an explicit in-code TODO to generalize this once other resource types need fulfillment.
- On `payment.failed`, only `payments.status` is set to `FAILED` — `orders.status` and any linked `subscriptions.status` are left untouched (order stays `PENDING`, allowing retry until `expires_at`).

## 5. Full Cross-Domain Relationship Summary

| From | To | Cardinality | Via |
|---|---|---|---|
| `users` | `refresh_tokens`, `users_roles`, `questions`, `assessments`, `assessments_participants`, `assessment_attempts`, `subscriptions`, `user_feature_usage`, `orders` | 1 → many | direct FK on each child table |
| `roles` | `users_roles`, `role_permissions` | 1 → many | direct FK |
| `resources` | `role_permissions` | 1 → many | direct FK |
| `assessments` | `assessments_questions`, `assessments_participants`, `assessment_attempts` | 1 → many | direct FK |
| `questions` | `assessments_questions` | 1 → many | direct FK |
| `assessment_attempts` | `assessment_attempt_responses` | 1 → many | direct FK |
| `assessments_questions` | `assessment_attempt_responses` | 1 → many | direct FK |
| `plans` | `plan_versions` | 1 → many | direct FK |
| `plan_versions` | `plan_features`, `subscriptions` | 1 → many | direct FK |
| `plan_features` | `user_feature_usage` | 1 → many | direct FK |
| `orders` | `payments` | 1 → many | direct FK |
| `subscriptions` | `orders` | 1 → 0..1 (per active checkout) | **application-level only** — `orders.resource_id` when `resource_type = SUBSCRIPTION`, no DB FK |
