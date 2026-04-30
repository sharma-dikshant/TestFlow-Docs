# Entity Relationship Diagram (ERD)

In this section, we will create an Entity Relationship Diagram (ERD) to visually represent the relationships between the entities in our database. An ERD is a graphical representation of the entities and their relationships in a database.

## v1

```mermaid
erDiagram

    users {
        uuid id PK
        varchar full_name
        varchar first_name
        varchar last_name
        varchar email
        varchar password
        varchar phone_number
        enum status
        boolean is_verified
        uuid role_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    roles {
        uuid id PK
        varchar name
        timestamp created_at
        timestamp deleted_at
    }

    actions {
        uuid id PK
        varchar name
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    resources {
        uuid id PK
        varchar name
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    permissions {
        uuid id PK
        uuid resource_id FK
        uuid action_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    roles_permissions {
        uuid id PK
        uuid role_id FK
        uuid permission_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    tests {
        uuid id PK
        varchar title
        varchar description
        timestamp start_time
        timestamp end_time
        int duration
        timestamp created_by FK
        timestamp created_at
    }

    questions {
        uuid id PK
        text question_text
        varchar type
        jsonb options
        jsonb test_cases
        int default_marks
    }

    test_questions {
        uuid id PK
        uuid test_id FK
        uuid question_id FK
        int marks
        int order_index
    }

    test_participants {
        uuid id PK
        uuid user_id FK
        uuid test_id FK
        varchar status
        timestamp started_at
        timestamp completed_at
    }

    submissions {
        uuid id PK
        uuid participant_id FK
        uuid test_question_id FK
        text answer
        jsonb evaluated_result
        int score
        timestamp submitted_at
    }

    %% Auth relationships
    users }o--|| roles : "role_id"
    roles ||--o{ roles_permissions : "id -> role_id"
    permissions ||--o{ roles_permissions : "id -> permission_id"
    actions ||--o{ permissions : "id -> action_id"
    resources ||--o{ permissions : "id -> resource_id"

    %% Test relationships
    users ||--o{ tests : creates
    users ||--o{ test_participants : participates

    tests ||--o{ test_participants : has
    tests ||--o{ test_questions : includes

    questions ||--o{ test_questions : mapped_in

    test_participants ||--o{ submissions : makes
    test_questions ||--o{ submissions : answered_in
```

## Authentication and Authorization

```mermaid
erDiagram
    users {
        uuid id PK
        varchar full_name
        varchar first_name
        varchar last_name
        varchar email
        varchar password
        varchar phone_number
        enum status
        boolean is_verified
        uuid role_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }
    roles {
        uuid id PK
        varchar name
        timestamp created_at
        timestamp deleted_at
    }
    actions {
        uuid id
        varchar name
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }
    resources {
        uuid id
        varchar name
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }
    permissions {
        uuid id PK
        uuid resource_id FK
        uuid action_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }
    roles_permissions {
        uuid id PK
        uuid role_id FK
        uuid permission_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    users }o--|| roles : "role_id"
    roles ||--o{ roles_permissions : "id -> role_id"
    permissions ||--o{ roles_permissions : "id -> permission_id"
    actions ||--o{ permissions : "id -> action_id"
    resources ||--o{ permissions : "id -> resource_id"
```

## Test Management

```mermaid
erDiagram

    USERS {
        int id PK
        string name
        string email
        string password
        datetime created_at
    }

    TESTS {
        int id PK
        string title
        string description
        int created_by FK
        datetime created_at
    }

    QUESTIONS {
        int id PK
        string question_text
        string type
        jsonb options
        jsonb test_cases
        int default_marks
    }

    TEST_QUESTIONS {
        int id PK
        int test_id FK
        int question_id FK
        int marks
        int order_index
    }

    TEST_PARTICIPANTS {
        int id PK
        int user_id FK
        int test_id FK
        string status
        datetime started_at
        datetime completed_at
    }

    SUBMISSIONS {
        int id PK
        int participant_id FK
        int test_question_id FK
        text answer
        jsonb evaluated_result
        int score
        datetime submitted_at
    }

    USERS ||--o{ TESTS : creates
    USERS ||--o{ TEST_PARTICIPANTS : participates

    TESTS ||--o{ TEST_PARTICIPANTS : has
    TESTS ||--o{ TEST_QUESTIONS : includes

    QUESTIONS ||--o{ TEST_QUESTIONS : mapped_in

    TEST_PARTICIPANTS ||--o{ SUBMISSIONS : makes
    QUESTIONS ||--o{ SUBMISSIONS : receives
```
