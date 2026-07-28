---
title: Entity Relationship Diagram (ERD) — Workspace & Organisation Model
description: Draft ERD exploring a workspace/organization-based redesign of the TestFlow schema — multi-tenant workspaces with workspace-scoped roles, owning assessments, subscriptions, and orders in place of today's direct-to-user model
status: workspace-draft
last_updated: 2026-07-28
---

# Entity Relationship Diagram (ERD) — Workspace & Organisation Model

This is a **draft**, not the current implemented schema — see [erd.md](erd.md) for that. It sketches out a workspace/organization layer sitting between `users` and the rest of the schema: users would join one or more `workspaces`, hold a `workspace_role` scoped to that workspace, and workspaces — rather than individual users — would own assessments, subscriptions, and orders. System-level roles/permissions stay as a separate, workspace-independent tier for platform-wide access (e.g. super admins).

## 1. User Identity & System-Level Role Management

Core user accounts plus a global (non-workspace-scoped) role/permission system for platform-wide access control.

```mermaid
erDiagram

    users {
        int id PK
        varchar full_name "200"
        varchar first_name "100"
        varchar last_name "100"
        varchar username "UNIQUE, 100"
        varchar email "UNIQUE, 255"
        varchar password "nullable"
        varchar phone_number "UNIQUE, nullable"
        boolean is_active
        boolean is_verified
        varchar oauth_provider "nullable"
        varchar oauth_id "nullable"
        timestamp last_login_at
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    refresh_tokens {
        int id PK
        int user_id FK
        varchar token_hash
        timestamp expires_at
        timestamp created_at
        timestamp deleted_at
    }

    system_roles {
        int id PK
        varchar key "UNIQUE"
        varchar display_name
        text description
        boolean is_system
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    system_permissions {
        int id PK
        varchar key "UNIQUE"
        varchar display_name
        text description
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    system_role_permissions {
        int id PK
        int role_id FK
        int permission_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    user_system_roles {
        int id PK
        int user_id FK
        int role_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    users ||--o{ refresh_tokens : issues
    users ||--o{ user_system_roles : assigned
    system_roles ||--o{ user_system_roles : contains
    system_roles ||--o{ system_role_permissions : grants
    system_permissions ||--o{ system_role_permissions : includes

```


## 2. Workspace Management

Introduces the multi-tenant boundary itself: `workspaces` (personal or organization), membership, and a workspace-scoped role/permission system distinct from the system-level one above.

```mermaid
erDiagram

    users { }

    workspaces {
        int id PK
        varchar name
        varchar slug "UNIQUE"
        varchar type "ENUM: PERSONAL, ORGANIZATION"
        varchar status "ENUM: ACTIVE, ARCHIVED"
        int owner_user_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    workspace_members {
        int id PK
        int workspace_id FK
        int user_id FK
        int workspace_role_id FK
        varchar status "ENUM: ACTIVE, INVITED, SUSPENDED"
        timestamp joined_at
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    workspace_roles {
        int id PK
        int workspace_id FK "NULL = system role"
        varchar name
        varchar key
        boolean is_system
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    workspace_permissions {
        int id PK
        varchar key
        varchar display_name
        text description
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    workspace_role_permissions {
        int id PK
        int workspace_role_id FK
        int workspace_permission_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    users ||--|| workspaces : owns_personal
    users ||--o{ workspace_members : joins

    workspaces ||--o{ workspace_members : has
    workspaces ||--o{ workspace_roles : defines

    workspace_roles ||--o{ workspace_members : assigned
    workspace_roles ||--o{ workspace_role_permissions : grants

    workspace_permissions ||--o{ workspace_role_permissions : contains
```


## 3. Assessment Engine

Same question/attempt lifecycle as the current schema, but reparented under `workspaces` — assessments belong to a workspace, and invitations replace direct participant assignment.

```mermaid
erDiagram

    questions {
        int id PK
        text question_text
        jsonb options
        int correct_option_id
        int created_by FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    assessments {
        int id PK
        int workspace_id FK
        varchar title
        text description
        timestamp start_time
        timestamp end_time
        int duration
        int max_participants
        decimal correct_marks
        decimal incorrect_marks
        varchar generation_status
        varchar status "DRAFT, PUBLISHED, ARCHIVED"
        int created_by FK
        timestamp published_at
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    assessment_questions {
        int id PK
        int assessment_id FK
        int question_id FK
        text question_text
        jsonb options
        int correct_option_id
        decimal correct_marks
        decimal incorrect_marks
        int order_index
        timestamp created_at
    }

    assessment_invitations {
        int id PK
        int assessment_id FK
        varchar email
        int invited_user_id FK "nullable"
        varchar status "PENDING, ACCEPTED, EXPIRED, REVOKED"
        timestamp expires_at
        int invited_by FK
        timestamp created_at
    }

    assessment_attempts {
        int id PK
        uuid uid
        int assessment_id FK
        int invitation_id FK
        int user_id FK
        decimal score
        int total_questions
        int attempted_questions
        int correct_responses
        int incorrect_responses
        varchar status
        timestamp started_at
        timestamp submitted_at
        timestamp created_at
        timestamp updated_at
    }

    assessment_attempt_answers {
        int id PK
        int attempt_id FK
        int assessment_question_id FK
        int selected_option_id
        boolean is_correct
        decimal marks_awarded
        timestamp answered_at
    }

    workspaces ||--o{ assessments : owns

    users ||--o{ questions : creates
    users ||--o{ assessments : creates

    assessments ||--o{ assessment_questions : contains
    questions ||--o{ assessment_questions : snapshots

    assessments ||--o{ assessment_invitations : sends

    assessment_invitations ||--o| assessment_attempts : starts

    users ||--o{ assessment_attempts : makes

    assessment_attempts ||--o{ assessment_attempt_answers : contains
    assessment_questions ||--o{ assessment_attempt_answers : answers
```


## 4. Commerce

Plans/features/subscriptions carried over from the current model, but subscription ownership and feature-usage tracking move from `users` to `workspaces`.

```mermaid
erDiagram

    plans {
        int id PK
        varchar key "UNIQUE"
        varchar display_name
        text description
        boolean is_active
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    plan_versions {
        int id PK
        int plan_id FK
        int version
        decimal price
        varchar currency
        varchar billing_cycle
        boolean is_current
        timestamp effective_from
        timestamp effective_until
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    features {
        int id PK
        varchar key "UNIQUE"
        varchar display_name
        text description
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    plan_features {
        int id PK
        int plan_version_id FK
        int feature_id FK
        varchar feature_value
        int reset_period_days
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    subscriptions {
        int id PK
        int workspace_id FK
        int plan_version_id FK
        varchar status
        timestamp starts_at
        timestamp expires_at
        timestamp cancelled_at
        boolean auto_renew
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    workspace_feature_usage {
        int id PK
        int workspace_id FK
        int plan_feature_id FK
        int used
        date period_start
        date period_end
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    workspaces ||--o{ subscriptions : owns

    plans ||--o{ plan_versions : versions

    plan_versions ||--o{ plan_features : defines

    features ||--o{ plan_features : contains

    plan_versions ||--o{ subscriptions : purchased

    plan_features ||--o{ workspace_feature_usage : tracked

    workspaces ||--o{ workspace_feature_usage : consumes
```

## 5. Payments & Orders

Orders move from user-owned to workspace-owned, with the acting user retained via `created_by` for audit purposes. Payments and webhook events are otherwise unchanged in shape from the current schema.

```mermaid
erDiagram

    orders {
        int id PK

        int workspace_id FK

        int created_by FK

        varchar resource_type "SUBSCRIPTION"

        int resource_id "nullable"

        decimal total_amount

        varchar currency

        varchar provider

        varchar status "PENDING, PAID, FAILED, CANCELLED, EXPIRED"

        jsonb metadata

        timestamp expires_at

        timestamp created_at

        timestamp updated_at
    }

    payments {
        int id PK

        int order_id FK

        decimal amount

        varchar currency

        varchar provider

        varchar provider_order_id

        varchar provider_payment_id

        varchar status "PENDING, SUCCESS, FAILED, REFUNDED"

        text failure_reason

        jsonb gateway_response

        timestamp paid_at

        timestamp created_at

        timestamp updated_at
    }

    webhook_events {
        int id PK

        varchar provider

        varchar event_id

        varchar event_type

        jsonb payload

        boolean processed

        timestamp processed_at

        timestamp created_at
    }

    workspaces ||--o{ orders : places

    users ||--o{ orders : created_by

    orders ||--o{ payments : has
```


## 6. Async Jobs

Unchanged from the current schema — the async job queue has no ownership model to migrate, since it carries no FK relations today.

```mermaid
erDiagram

    jobs {
        bigint id PK "returned to clients as a string"
        varchar type "100 — ENUM-like, currently only CREATE_ASSESSMENT is issued"
        varchar queue "50 — plain string, e.g. 'assessment'"
        varchar status "ENUM: PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED — default PENDING"
        jsonb payload "nullable — arbitrary job input, e.g. {assessmentId, userId, prompt, count}"
        jsonb result "nullable — set on completion"
        jsonb error "nullable — set on failure"
        int attempts "default 0"
        int max_attempts
        text status_description "nullable — human-readable progress message"
        timestamp created_at
        timestamp started_at "nullable"
        timestamp completed_at "nullable"
    }
```
