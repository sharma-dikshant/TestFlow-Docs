---
title: Architecture
description: Current-state system architecture for the TestFlow backend — modules, request pipeline, and cross-module call chains
status: stable
last_updated: 2026-07-15
---

# Architecture

This page reflects the **actual implemented architecture** as of 2026-07-15 (verified directly against source, not aspirational design). Where the implementation diverges from what you might expect (e.g. no AI question generation yet, no Redis/OTP/OAuth), it is called out explicitly.

## 1. Module Map

`AppModule` wires together 10 feature modules plus `DatabaseModule` and a global `ConfigModule`:

```mermaid
flowchart TD
    subgraph core["Core"]
        config["ConfigModule (global)"]
        database["DatabaseModule\n(TypeORM + Postgres)"]
    end

    subgraph identity["Identity"]
        users["UsersModule"]
        auth["AuthModule"]
        rbac["RbacModule"]
    end

    subgraph testing["Assessment Engine"]
        questions["QuestionsModule"]
        assessments["AssessmentsModule"]
        attempts["AttemptsModule"]
    end

    subgraph commerce["Monetization"]
        commerceMod["CommerceModule\n(plans · subscriptions · usage)"]
        payments["PaymentsModule\n(+ RazorpayService)"]
        webhooks["WebhooksModule"]
    end

    auth --> users
    attempts --> assessments
    assessments --> questions
    payments --> commerceMod
    webhooks --> payments

    core -.-> identity
    core -.-> testing
    core -.-> commerce
```

**Import graph (who depends on whom):**

| Module              | Imports                                                                                                                                                         | Exports                                                                     |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `AuthModule`        | `UsersModule`, `JwtModule.register({})`, `RefreshToken` repo                                                                                                    | `AuthService`, `JwtAuthGuard`, `JwtModule`                                  |
| `UsersModule`       | `User` repo, `RbacModule`                                                                                                                                       | `UsersService`                                                              |
| `RbacModule`        | `Role`/`Resource`/`RolePermission`/`UserRole` repos                                                                                                             | `AccessControlService`, `RolesGuard`, `PermissionGuard`, `UserRolesService` |
| `QuestionsModule`   | `Question` repo                                                                                                                                                 | `QuestionsService`                                                          |
| `AssessmentsModule` | `AuthModule`, `QuestionsModule`, assessment repos                                                                                                               | `AssessmentsService`                                                        |
| `AttemptsModule`    | `AuthModule`, `AssessmentsModule`, attempt repos                                                                                                                | `AttemptsService`                                                           |
| `CommerceModule`    | `Plan`/`PlanVersion`/`PlanVersionFeature`/`Subscription`/`UserFeatureUsage` repos (single flat module — plans/subscriptions/usage are folders, not sub-modules) | _(none)_                                                                    |
| `PaymentsModule`    | `Payment`/`Order` repos                                                                                                                                         | `PaymentsService`, `RazorpayService`                                        |
| `WebhooksModule`    | `WebhookEvent` repo, `PaymentsModule`                                                                                                                           | _(none)_                                                                    |

`CommerceModule` and `PaymentsModule` are not linked by a NestJS `imports`/`exports` relationship — `PaymentsService` reaches directly into the `Subscription` entity (cross-module entity import) to flip subscription status on payment capture. This is a known, explicitly TODO-flagged shortcut — see [Commerce & Payments HLD](hld-design/commerce.md#cross-module-coupling).

## 2. Request Pipeline

Every request passes through the same pipeline, wired partly in `app.module.ts` (guards) and partly imperatively in `main.ts` (filters/interceptors/validation) — the filter/interceptor wiring is **not** done via `APP_FILTER`/`APP_INTERCEPTOR` DI tokens, so it won't show up in the Nest module graph.

```mermaid
flowchart LR
    req["Incoming HTTP request"]
    vpipe["Global ValidationPipe\nwhitelist + forbidNonWhitelisted + transform"]
    g1["JwtAuthGuard\n(skips if @Public())"]
    g2["RolesGuard\n(no-op unless @Roles() present)"]
    g3["PermissionGuard\n(no-op unless @Permissions() present)"]
    handler["Controller handler"]
    respI["ResponseInterceptor\n(wraps ApiResponse → envelope)"]
    serI["ClassSerializerInterceptor\n(entity → plain object)"]
    filter["HttpExceptionFilter\n(catches all thrown errors)"]
    res["HTTP response"]

    req --> vpipe --> g1 --> g2 --> g3 --> handler
    handler -->|"success"| respI --> serI --> res
    handler -->|"throws"| filter --> res
```

**Guard chain detail** — all three guards are registered globally via `APP_GUARD` in `app.module.ts`, in this order:

1. **`JwtAuthGuard`** — reads `IS_PUBLIC_KEY` metadata (`@Public()` decorator); if absent, requires a valid `Authorization: Bearer <token>` verified against `JWT_ACCESS_SECRET`, and attaches `request.user = { sub, email, username }`.
2. **`RolesGuard`** — reads `@Roles()` metadata; currently unused by any controller in the codebase (present but a no-op pass-through everywhere today).
3. **`PermissionGuard`** — reads `@Permissions()` metadata (e.g. `'questions:create'`); checks `AccessControlService.userHasPermission(userId, permission)`, which allows if the user's role has that exact resource key **or** the wildcard `*:*` resource.

Only the `questions`, `rbac`, and `users` (role-assignment) controllers actually apply `@Permissions()`. `assessments` and `attempts` controllers rely on `JwtAuthGuard` alone plus service-level ownership checks (e.g. `assessment.createdBy === userId`).

## 3. Response Envelope

The response envelope actually implemented differs from earlier drafts of this doc — see `src/common/interceptors/response.interceptor.ts` and `src/common/filters/http-exception.filter.ts` for ground truth. Full shape is documented in [API Design](api-design/v1.md#response-format).

Success: `{ success: true, message: string, data: T, meta?: object }` — no `error` key exists anywhere in the codebase.
Error: `{ success: false, message: string, ...extraFields }` — extra fields are spread flat, not nested under `error`.

## 4. Assessment Engine Flow

The core product flow: create/generate an assessment → a user registers as a participant → starts an attempt → answers questions → submits → gets an async-evaluated result.

```mermaid
flowchart TD
    user["User"]

    subgraph assessmentsMod["AssessmentsModule"]
        assessCtrl["AssessmentsController"]
        assessSvc["AssessmentsService"]
    end

    subgraph attemptsMod["AttemptsModule"]
        attemptCtrl["AttemptsController"]
        attemptSvc["AttemptsService"]
        evalSvc["EvaluationService"]
    end

    subgraph questionsMod["QuestionsModule"]
        questionSvc["QuestionsService"]
    end

    db[("PostgreSQL")]

    user -->|"POST /assessments\nor /assessments/generate"| assessCtrl
    assessCtrl --> assessSvc
    assessSvc -->|"generate(): pull N random questions\n(NOT LLM-backed — see note below)"| questionSvc
    assessSvc -->|"snapshot questions + create participant row"| db

    user -->|"POST /assessments/:id/register"| assessCtrl
    user -->|"POST /attempts/:assessmentId/start"| attemptCtrl
    attemptCtrl --> attemptSvc
    attemptSvc -->|"read assessment/question/participant data"| assessSvc
    attemptSvc -->|"create attempt row"| db

    user -->|"PUT /attempts/:uid/responses/:qid"| attemptCtrl
    attemptSvc -->|"compute isCorrect + marksAwarded\nagainst snapshotted correctOptionId"| db

    user -->|"POST /attempts/:uid/submit"| attemptCtrl
    attemptSvc -->|"mark SUBMITTED, mark participant COMPLETED"| db
    attemptSvc -.->|"fire-and-forget, unawaited"| evalSvc
    evalSvc -->|"aggregate responses → score,\nstatus = EVALUATED"| db

    user -->|"GET /attempts/:uid/result"| attemptCtrl
```

**Important implementation notes:**

- **"Generate" is not AI-powered today.** `POST /assessments/generate` is documented/named as AI-powered question generation, but `AssessmentsService.generate()` actually calls `QuestionsService.findRandom(n)` — a plain `ORDER BY RANDOM() LIMIT n` pull from the existing question bank. An unused scaffold exists at `src/ai/interfaces/generated-question.interface.ts` (`GeneratedQuestion`/`GeneratedQuestionOption`) with zero consumers anywhere in the codebase — it appears to be prepared for a future LLM integration that hasn't been wired up yet.
- **Scoring happens at answer-time, not submit-time.** `AttemptsService.upsertResponse()` computes `isCorrect` and `marksAwarded` (including negative marking) the moment a response is saved, using the per-question `correctMarks`/`incorrectMarks` snapshotted onto `AssessmentQuestion` when questions were added to the assessment. `EvaluationService.evaluate()` only aggregates already-computed response rows into the attempt-level `score`/`attemptedQuestions`/`correctResponses`/`incorrectResponses` and flips status to `EVALUATED`.
- **Evaluation is asynchronous and unawaited.** `submitAttempt()` returns `true` to the client immediately after flipping status to `SUBMITTED`; `EvaluationService.evaluate()` runs afterward without being awaited. There is a narrow window where `GET /attempts/:uid/result` returns 400 (`"Attempt has not been evaluated yet"`) right after submit.
- **Legacy/dead code**: `src/assessments/dto/attempt-response.dto.ts`, `get-my-attempts-query.dto.ts`, `submit-response.dto.ts`, and `src/assessments/enums/attempt-status.enum.ts` are unused duplicates of the real files in `src/attempts/*` — left over from before the attempt/submission flow was extracted into its own module. `AttemptsModule` is the sole owner of the attempt/submission/evaluation flow today.

## 5. Commerce & Payments Flow

Checkout, payment, and webhook-driven fulfillment span three modules and are **not** wired through an event bus — fulfillment is a direct, synchronous cross-module call.

```mermaid
sequenceDiagram
    autonumber

    actor U as User
    participant Sub as SubscriptionsService
    participant Pay as PaymentsService
    participant Rzp as RazorpayService
    participant Api as Razorpay
    participant Hook as WebhooksService

    U->>Sub: POST /subscriptions/checkout
    Sub->>Sub: Validate plan
    Sub->>Sub: Check existing subscription
    Sub->>Sub: Create Subscription PENDING
    Sub->>Sub: Create Order PENDING
    Sub-->>U: Return subscription and order

    U->>Pay: POST /payments/razorpay/create
    Pay->>Pay: Validate order
    Pay->>Rzp: createOrder
    Rzp->>Api: orders.create
    Api-->>Rzp: Razorpay order
    Rzp-->>Pay: Razorpay order
    Pay->>Pay: Create Payment PENDING
    Pay-->>U: Return Razorpay checkout data

    Api-->>Hook: payment.captured webhook
    Hook->>Rzp: Verify signature
    Hook->>Hook: Deduplicate event
    Hook->>Pay: markPaymentCaptured
    Pay->>Pay: Payment SUCCESS
    Pay->>Pay: Order PAID
    Pay->>Pay: Activate Subscription
    Hook->>Hook: Mark event processed
```

See [Commerce, Payments & Webhooks HLD](hld-design/commerce.md) for the full endpoint/entity/edge-case breakdown, including gaps (payment failure doesn't cascade to Order/Subscription status, `AI_CREDITS` order fulfillment has no handler yet, no renewal/cancellation/expiry logic exists for `PAST_DUE`/`CANCELLED`/`EXPIRED` subscription states).

## 6. What's Deliberately Not in the Stack

To avoid re-introducing assumptions from earlier design drafts, note what is **not** present despite being referenced in older docs or `.env.example`:

| Not present               | Where it might be assumed to exist                                                                                                                      |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Redis                     | Auth token storage/OTP/lockout (older `auth.md` draft) — refresh tokens are actually DB-backed (`refresh_tokens` table, SHA-256 hash)                   |
| OTP module                | Pre-registration email verification (older `auth.md` draft) — signup has no verification step today                                                     |
| OAuth flow                | `oauth_provider`/`oauth_id` columns exist on `users` but no OAuth controller/service/strategy exists anywhere                                           |
| MailService / SMTP        | No email sending capability in the codebase at all                                                                                                      |
| `helmet`, `cookie-parser` | Not dependencies; CORS is wide-open (`origin: true, credentials: true`); auth is pure Bearer token, no cookies                                          |
| Passport strategies       | `passport`/`passport-jwt` are installed but unused — `JwtAuthGuard` hand-verifies tokens via `JwtService.verify()` directly                             |
| Stripe integration        | `PaymentProvider.STRIPE` exists as an enum value on `payments`/`orders` but there is no Stripe SDK dependency or service — only Razorpay is implemented |
| Redis/queue env vars      | `REDIS_HOST`, `REDIS_PORT`, `NODE_ENV`, `LOG_LEVEL` are declared in `.env.example` but never read anywhere in `src/`                                    |

## 7. Related Pages

- [ERD](erd.md) — full schema
- [API Design v1](api-design/v1.md) — full endpoint reference
- [Data Access Patterns](data-access-pattern.md) — query patterns per module
- [HLD — Authentication](hld-design/auth.md)
- [HLD — RBAC](hld-design/rbac.md)
- [HLD — Assessments, Attempts & Questions](hld-design/assessments.md)
- [HLD — Commerce, Payments & Webhooks](hld-design/commerce.md)
