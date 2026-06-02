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
        varchar oauth_id "UNIQUE (oauth_provider, oauth_id)"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    roles {
        int id PK
        varchar name "UNIQUE"
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
        varchar key "UNIQUE (e.g. tests:create)"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    permissions {
        int id PK
        int role_id FK "FK → roles.id"
        int resource_id FK "FK → resources.id"
        text description
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    tests {
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
        varchar type "ENUM: MCQ, CODING, SUBJECTIVE"
        jsonb options
        jsonb test_cases
        int default_marks
        int created_by FK "FK → users.id"
        timestamp created_at
        timestamp updated_at
    }

    tests_questions {
        int id PK
        int test_id FK "FK → tests.id"
        int question_id FK "FK → questions.id"
        int marks
        int negative_marks "marks deducted for wrong answer"
        int order_index
    }

    tests_participants {
        int id PK
        int user_id FK "FK → users.id"
        int test_id FK "FK → tests.id"
        varchar status "ENUM: NOT_STARTED, IN_PROGRESS, COMPLETED"
        timestamp started_at
        timestamp completed_at
        timestamp created_at
        timestamp deleted_at
    }

    submissions {
        int id PK
        int participant_id FK "FK → tests_participants.id"
        int test_question_id FK "FK → tests_questions.id"
        text answer
        jsonb evaluated_result
        int score
        int attempt_no
        timestamp submitted_at
    }

    %% Auth relationships
    users ||--o{ users_roles : has
    roles ||--o{ users_roles : assigned_to

    roles ||--o{ permissions : has
    resources ||--o{ permissions : has

    %% Test relationships
    users ||--o{ tests : creates
    users ||--o{ questions : creates
    users ||--o{ tests_participants : participates

    tests ||--o{ tests_participants : has
    tests ||--o{ tests_questions : includes

    questions ||--o{ tests_questions : mapped_in

    tests_participants ||--o{ submissions : makes
    tests_questions ||--o{ submissions : answered_in

```
