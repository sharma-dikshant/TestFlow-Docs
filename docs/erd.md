---
title: Entity Relationship Diagram (ERD)
description: Full ERD version history for the TestFlow database schema, from v1 through v2.2.1
status: stable
last_updated: 2026-05-05
---

# Entity Relationship Diagram (ERD)

This page documents the evolution of the TestFlow database schema. The most recent version is v2.2.1.

## Latest Version

```mermaid
erDiagram

    users {
        int id PK
        varchar full_name
        varchar first_name
        varchar last_name
        varchar username "UNIQUE"
        varchar email "UNIQUE"
        varchar password
        varchar phone_number "UNIQUE"
        boolean is_active
        boolean is_verified
        varchar oauth_provider "ENUM: GOOGLE, GITHUB, NULL"
        varchar oauth_id
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    roles {
        int id PK
        varchar name "UNIQUE"
        varchar key "UNIQUE"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    users_roles {
        int id PK
        int user_id FK "FK → users.id"
        int role_id FK "FK → roles.id"
        timestamp created_at
    }

    resources {
        int id PK
        varchar key "UNIQUE (e.g. assessments:create)"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    role_permissions {
        int id PK
        int role_id FK "FK → roles.id"
        int resource_id FK "FK → resources.id"
        text description
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    assessments {
        int id PK
        varchar title
        text description
        timestamp start_time
        timestamp end_time
        int duration "in minutes"
        int max_participants
        int current_participants
        int created_by FK "FK → users.id"
        timestamp created_at
        timestamp updated_at
    }

    questions {
        int id PK
        text question_text
        jsonb options
        int correct_option_id
        int created_by FK "FK → users.id"
        timestamp created_at
        timestamp updated_at
    }

    assessments_questions {
        int id PK
        int assessment_id FK "FK → assessments.id"
        int question_id FK "FK → questions.id"

        text question_text
        jsonb options
        int correct_option_id

        decimal correct_marks
        decimal incorrect_marks

        int order_index

        timestamp created_at
    }

    assessments_participants {
        int id PK
        int user_id FK "FK → users.id"
        int assessment_id FK "FK → assessments.id"
        varchar status "ENUM: NOT_STARTED, IN_PROGRESS, COMPLETED"
        timestamp started_at
        timestamp completed_at
        timestamp created_at
        timestamp deleted_at
    }


    assessment_attempts {
        int id PK
        int assessment_id FK "FK → assessments.id"
        int user_id FK "FK → users.id"

        timestamp started_at
        timestamp submitted_at

        decimal score
        int total_questions
        int attempted_questions
        int correct_responses
        int incorrect_responses

        varchar status "ENUM: IN_PROGRESS, SUBMITTED, EVALUATED"

        timestamp created_at
        timestamp updated_at
    }

    assessment_attempt_responses {
        int id PK
        int attempt_id FK "FK → assessment_attempts.id"
        int assessment_question_id FK "FK → assessments_questions.id"

        int selected_option_id

        boolean is_correct
        decimal marks_awarded

        timestamp answered_at
    }

    %% Auth relationships
    users ||--o{ users_roles : has
    roles ||--o{ users_roles : assigned_to

    roles ||--o{ role_permissions : has
    resources ||--o{ role_permissions : grants

    %% Question bank relationships
    users ||--o{ questions : creates

    %% assessment relationships
    users ||--o{ assessments : creates

    assessments ||--o{ assessments_questions : includes
    questions ||--o{ assessments_questions : mapped_in

    %% Participant relationships
    users ||--o{ assessments_participants : participates
    assessments ||--o{ assessments_participants : has



    %% Attempt relationships
    users ||--o{ assessment_attempts : makes
    assessments ||--o{ assessment_attempts : attempted

    assessment_attempts ||--o{ assessment_attempt_responses : contains
    assessments_questions ||--o{ assessment_attempt_responses : answered_for

```

## Subscription Design

This diagram scopes the subscription and billing subsystem: plan versioning, active subscriptions, orders/payments, and per-feature usage tracking. `users` is shown trimmed to the fields relevant here; see [Latest Version](#latest-version) above for the full entity.

```mermaid
erDiagram

    users {
        int id PK
        varchar full_name
        varchar email
        timestamp created_at
    }

    plans {
        int id PK
        varchar name "ENUM: FREE, PRO"
        varchar code "UNIQUE"
        text description
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }

    plan_versions {
        int id PK
        int plan_id FK "FK → plans.id"
        int version "UNIQUE per plan"
        decimal price
        varchar currency
        varchar billing_cycle "ENUM: MONTHLY, YEARLY"
        boolean is_current "Only one current version per plan"
        timestamp effective_from
        timestamp effective_until
        timestamp created_at
        timestamp updated_at
    }

    plan_features {
        int id PK
        int plan_version_id FK "FK → plan_versions.id"
        varchar feature_name
        varchar feature_value
        int reset_period "Number of days before reset"
        timestamp created_at
    }

    subscriptions {
        int id PK
        int user_id FK "FK → users.id"
        int plan_version_id FK "FK → plan_versions.id"
        varchar status "ENUM: PENDING, ACTIVE, PAST_DUE, CANCELLED, EXPIRED"
        timestamp starts_at
        timestamp expires_at
        timestamp cancelled_at
        boolean auto_renew
        timestamp created_at
        timestamp updated_at
    }

    payments {
        int id PK
        int subscription_id FK "FK → subscriptions.id"
        decimal amount
        varchar currency
        varchar provider "ENUM: STRIPE, RAZORPAY"
        varchar provider_order_id
        varchar provider_payment_id
        varchar status "ENUM: PENDING, SUCCESS, FAILED, REFUNDED"
        text failure_reason
        timestamp paid_at
        timestamp created_at
        timestamp updated_at
    }

    user_feature_usage {
        int id PK
        int user_id FK "FK → users.id"
        int plan_feature_id FK "FK → plan_features.id"
        int used
        date period_start
        date period_end
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    webhook_events {
        int id PK
        varchar provider
        varchar event_id "UNIQUE"
        varchar event_type
        jsonb payload
        boolean processed
        timestamp processed_at
        timestamp created_at
    }

    %% Subscription relationships
    users ||--o{ subscriptions : purchases
    plans ||--o{ plan_versions : has_versions
    plan_versions ||--o{ plan_features : defines
    plan_versions ||--o{ subscriptions : purchased_as

    %% Billing relationships
    users ||--o{ subscriptions : purchases
    subscriptions ||--o{ payments : payments

    %% Usage relationships
    users ||--o{ user_feature_usage : consumes
    plan_features ||--o{ user_feature_usage : tracked_by

```