---
title: Commerce, Payments and Webhooks
description: How TestFlow models plans and feature limits, sells subscriptions through Razorpay, and meters AI usage.
status: stable
last_updated: 2026-10-27
---

# Commerce, Payments and Webhooks

## 1. What this page covers

"Commerce" is everything to do with money and limits:

- **Plans**: what we sell (price, billing cycle, and what you get).
- **Subscriptions**: which plan a user or an organization is on right now.
- **Payments**: taking money through **Razorpay** (an Indian payment gateway) and confirming it with a **webhook** (a server-to-server HTTP call Razorpay makes to us).
- **Usage**: counting how much of a limited feature (for example, AI assessment generations) someone has used, and refusing when they run out.
- **Admin**: endpoints that platform admins use to manage the plan catalog and look at payments, subscriptions, and usage.

The code is split into three NestJS modules:

| Module | Folder | Owns |
|---|---|---|
| `CommerceModule` | `src/commerce/` | Plans, subscriptions, usage, admin endpoints. Exports `UsageService` and `SubscriptionsService` to other modules. |
| `PaymentsModule` | `src/commerce/payments/` | Orders, payments, the Razorpay client. Exports `PaymentsService` and `RazorpayService`. |
| `WebhooksModule` | `src/webhooks/` | The public `POST /webhooks/razorpay` endpoint. Imports `PaymentsModule`. |

```mermaid
flowchart LR
    auth["Identity / Auth"] -->|"createFreeSubscription"| commerce
    orgs["Organizations"] -->|"createOrgTrialSubscription"| commerce
    assess["Assessment generation"] -->|"consume / release / getFeatureValue"| commerce
    roadmaps["Learning roadmaps"] -->|"consume"| commerce
    ai["AI model selector"] -->|"getFeatureValue"| commerce
    commerce["CommerceModule"]
    webhooks["WebhooksModule"] --> payments["PaymentsModule"]
    payments -.->|"writes Subscription rows directly"| commerce
```

The dotted arrow is a shortcut: `PaymentsService` imports the `Subscription` entity and updates it itself, instead of calling `SubscriptionsService`. There is a `TODO` about this in `src/commerce/payments/payments.service.ts`.

## 2. Key terms

| Term | Meaning |
|---|---|
| **Feature** | A thing a plan can grant, identified by a fixed `FeatureKey` (table `features`). |
| **Plan** | A product we sell, e.g. `FREE` or `PRO` (table `plans`). Has a public `code`. |
| **Plan version** | One price point of a plan: price, currency, billing cycle (table `plan_versions`). |
| **Plan feature** | One feature attached to one plan version, with its value and reset period. Entity `PlanVersionFeature`, but the **table is `plan_features`**. |
| **Subscription** | A user's or an organization's link to one plan version, with a status and dates. |
| **Order** | "We expect this much money for this thing." Created at checkout. |
| **Payment** | One attempt to pay an order through a provider (Razorpay). An order can have several payments. |
| **Usage row** | A counter in `user_feature_usage`: how much of one plan feature was used in the current period. |
| **Audience** | Who can buy a plan: `individual` (for yourself) or `organization`. |

## 3. Plans, versions and features

### 3.1 How pricing and limits are modelled

```mermaid
erDiagram
    plans ||--o{ plan_versions : "has"
    plan_versions ||--o{ plan_features : "grants"
    features ||--o{ plan_features : "is used in"
    plan_versions ||--o{ subscriptions : "subscribed by"
    plan_features ||--o{ user_feature_usage : "counted in"
```

- **`plans`** (`src/common/entities/plan.entity.ts`): `name`, unique `code`, `description`, `audience`, `is_active`, `is_displayed`, `sort_order`.
    - `is_active = false` means it cannot be bought.
    - `is_displayed = false` hides it from the public list, but existing subscribers keep it.
    - `sort_order` sets the order of the public list (lowest first, then `id`).
- **`plan_versions`** (`plan-version.entity.ts`): `version` number, `price` (decimal 10,2), `currency` (3 letters), `billing_cycle` (`MONTHLY`, `YEARLY`, `ONE_TIME`), `is_current`, `effective_from`, `effective_until`, `expiry_days`.
    - `expiry_days` only applies to `ONE_TIME` versions. `null` means "never expires". The admin API refuses a value on a recurring version (`400`).
- **`plan_features`** (`plan-version-feature.entity.ts`): `feature_id`, `display_name`, `feature_value` (a string), `type`, `reset_period` (days, or `null`). A partial unique index allows one non-deleted row per (version, feature).
- **`features`** (`feature.entity.ts`): one row per `FeatureKey`. Seeded, not created through the API.

### 3.2 Feature types

A plan feature has a `type` (`PlanFeatureType` in `src/common/enums/commerce.enums.ts`) that says how to read `feature_value`:

| Type | `feature_value` means | Counted? |
|---|---|---|
| `limit` | A quota. A number like `"2"` is the limit. A non-number like `"unlimited"` means no limit. | Yes, by `UsageService.consume` |
| `enum` | A setting, e.g. a model code or a list of question types. | No. `consume` refuses an enum feature. |

### 3.3 The feature keys today

| `FeatureKey` | Type in practice | Used by | Effect |
|---|---|---|---|
| `AI_ASSESSMENT` | `limit` | Assessment generation | How many AI assessment generations per period. |
| `AI_LEARNING_ROADMAP` | `limit` | Learning roadmaps | How many AI roadmaps per period. |
| `AI_MODEL` | `enum` | `src/ai/ai-model-selector.service.ts` | Which LLM model the user's AI calls use. Missing means the default model. |
| `ALLOWED_QUESTION_TYPES` | `enum` | `src/assessments/generation/generation-entitlement.service.ts` | Comma-separated question types, e.g. `mcq,coding`. Missing or empty means `mcq` only (`src/commerce/plans/allowed-question-types.ts`). |

"Number of participants" and "number of assessments" are **not** metered features. Participant caps live on the assessment itself (see [Assessments](assessments.md)).

### 3.4 Why versions exist

Prices change, but people who already paid should keep what they bought. So instead of changing a plan's price in place, an admin adds a **new version**:

1. The old current version gets `is_current = false`, and `effective_until` is set (if it was empty) to the new version's `effective_from`.
2. The new version is inserted with `version = old + 1` and `is_current = true`.

Both steps run in one transaction (`CommerceAdminService.createPlanVersion` in `src/commerce/admin.service.ts`). A **partial unique index** (`UQ_plan_versions_plan_id_current`, `WHERE is_current = true`) makes sure a plan has at most one current version.

Existing subscriptions point at a `plan_version_id`, so they keep their old version's price and features.

Checkout asks for **both** `planCode` and `version`. So a client can buy an older version, as long as its `effective_until` has not passed.

### 3.5 Built-in plans and seeding

Two plans are special. Code finds them **by name**, so they must exist and be active (`src/commerce/plans/plan.constants.ts`):

| Name | Code (seeded) | Audience | Cycle | Price | Seeded feature | Who gets it |
|---|---|---|---|---|---|---|
| `FREE` | `FREEPLAN` | individual | `MONTHLY` | 0 USD | `AI_ASSESSMENT` = 2, reset every 1 day | Every user, when their account is verified |
| `ORG_TRIAL` | `ORGTRIAL` | organization | `ONE_TIME`, 7 days | 0 USD | `AI_ASSESSMENT` = 2, reset every 1 day | Every new organization |

- `npm run seed` runs `src/seeders/feature.seeder.ts` (creates the four features) and then `src/seeders/plan.seeder.ts` (creates each plan only if no plan with that name exists yet).
- Because code looks these plans up by name, admins **cannot rename or deactivate** `FREE` or `ORG_TRIAL` (`CommerceAdminService.updatePlan` returns `409`). Hiding, re-describing or re-ordering them is fine.
- Paid plans (for example `PRO`) are not seeded. Admins create them through the admin API (section 8).
- Admin-created plans get a random 10-character `code` from an alphabet without look-alike characters (no `I`, `O`, `0`, `1`).

## 4. Subscriptions

### 4.1 Personal vs organization subscriptions

Both exist, in the same `subscriptions` table:

| | Personal | Organization |
|---|---|---|
| `organization_id` | `NULL` | the org's id |
| `user_id` | the owner | the purchaser (the org creator for the trial) |
| How it starts | `FREE` at verification, or a paid plan through checkout | `ORG_TRIAL` when the org is created |
| Can be bought? | Yes, `individual` plans via `POST /v1/subscriptions/checkout` | **No**. There is no org checkout endpoint yet, and checkout refuses `organization` plans with `422`. |
| Usage | Counted per user | **Pooled**: one counter per organization and feature, shared by all members |

Which subscription applies depends on the `x-organization-id` request header (see [Organizations](organizations.md)). With the header, the org's plan is used, and the member's personal plan is never used as a fallback. Without it, the user's personal plan is used.

### 4.2 How a subscription is created

| Trigger | Code | Result |
|---|---|---|
| Email verification, new Google sign-in, or claiming an unverified account | `src/identity/auth/auth.service.ts` calls `SubscriptionsService.createFreeSubscription` | `FREE` current version, `ACTIVE`, `expires_at` = start + 1 month |
| A personal user with no active subscription spends usage | `UsageService.consume` calls `createFreeSubscription` | Same as above, created lazily inside the usage transaction |
| Organization created | `src/organizations/services/organizations.service.ts` calls `createOrgTrialSubscription` | `ORG_TRIAL`, `ACTIVE`, 7 days (or the version's `expiry_days`), `auto_renew = false` |
| User buys a plan | `SubscriptionsService.checkout` | `PENDING` subscription plus a `PENDING` order. Becomes `ACTIVE` after payment. |

`expires_at` comes from `computeExpiry` in `src/commerce/subscriptions/subscriptions.service.ts`: +1 month for `MONTHLY`, +1 year for `YEARLY`, `+expiry_days` for `ONE_TIME` (or `null` when it has no `expiry_days`).

### 4.3 Lifecycle

```mermaid
stateDiagram-v2
    [*] --> PENDING : checkout
    [*] --> ACTIVE : free plan or org trial
    PENDING --> ACTIVE : payment.captured webhook
    PENDING --> PENDING : payment failed, user retries
    ACTIVE --> SUSPENDED : user pays for another personal plan
    note right of SUSPENDED
        PAST_DUE, CANCELLED and EXPIRED
        exist in the enum but no code
        sets them today
    end note
```

What the code really does today:

- **Upgrade or switch plan**: the user checks out a different personal plan. When that payment is captured, every other `ACTIVE` personal subscription of that user (including `FREE`) is set to **`SUSPENDED`**, and the new one becomes `ACTIVE`. There is no proration and no refund of the old plan.
- **Downgrade**: works the same way. Buying a cheaper plan suspends the more expensive one.
- **Cancel**: no endpoint. `cancelled_at` is never written.
- **Expiry**: there is **no scheduler or job** that expires subscriptions (`src/schedulers/` only has the newsletter scheduler). Every "which plan is this user on" query checks `status = ACTIVE` only and ignores `expires_at`. So an `ACTIVE` subscription keeps working after `expires_at`, including the 7-day org trial.
- **Renewal**: `auto_renew` is stored (`true` for recurring, `false` for `ONE_TIME` and the trial), but nothing reads it. There is no recurring billing.
- A `PENDING` subscription whose order expired stays `PENDING` forever. Nothing cleans it up.

## 5. Checkout and payment with Razorpay

### 5.1 Why two steps

Checkout (`POST /v1/subscriptions/checkout`) **does not call Razorpay**. It only saves our own records: a `PENDING` subscription and a `PENDING` order. The Razorpay order is created by a second call, `POST /v1/payments/razorpay/create`.

This keeps "what is being bought" separate from "which payment provider is used". The order can be checked (owner, status, expiry) before any money is involved, and the provider step can be retried safely.

The order gets a placeholder `provider_order_id` of `chk_<uuid>` to fill its unique column. The real Razorpay order id is stored on the **payment** row, not on the order.

### 5.2 The full flow

```mermaid
sequenceDiagram
    autonumber
    participant C as Client app
    participant S as SubscriptionsController
    participant P as PaymentsController
    participant R as Razorpay
    participant W as WebhooksController
    participant DB as Postgres

    C->>S: POST /v1/subscriptions/checkout with planCode and version
    S->>DB: check plan, version, no ACTIVE sub for this plan
    S->>DB: insert Subscription PENDING and Order PENDING, 30 min expiry
    S-->>C: subscription and order
    C->>P: POST /v1/payments/razorpay/create with orderId
    P->>DB: check owner, status PENDING, not expired, latest payment
    P->>R: orders.create with amount x 100, currency, receipt
    R-->>P: razorpay order id
    P->>DB: insert Payment PENDING with providerOrderId
    P-->>C: razorpayOrderId and razorpayKeyId
    C->>R: open Razorpay Checkout widget and pay
    R->>W: POST /webhooks/razorpay with x-razorpay-signature
    W->>W: verify HMAC of raw body with RAZORPAY_WEBHOOK_SECRET
    W->>DB: look up webhook_events by eventId
    W->>DB: one transaction - Payment SUCCESS, Order PAID, old subs SUSPENDED, new sub ACTIVE
    W->>DB: save webhook_events row, processed true
    W-->>R: 200 OK
```

Step by step:

1. **Checkout** (`SubscriptionsService.checkout`, one transaction). It checks, in order:
    - plan exists by `code`, else `404`
    - plan `is_active`, else `422`
    - plan `audience` is `individual`, else `422` ("can only be bought for an organization")
    - the requested `version` exists, else `404`
    - `effective_until` has not passed, else `422`
    - the user has no `ACTIVE` personal subscription **to this same plan**, else `409`

    Then it saves the subscription (`PENDING`) and the order (`PENDING`, amount = version price, expires after `CHECKOUT_ORDER_TTL_MINUTES`, default 30).
2. **Create the Razorpay payment** (`PaymentsService.createRazorpayPayment`). It checks the order exists (`404`), belongs to the caller (`403`), is `PENDING` (`422`), and has not expired (`422`). Then it looks at the **latest** payment for the order:
    - `SUCCESS`: `409` "already been paid".
    - `PENDING` with a Razorpay order id: the same payment is **returned again**. This makes the call safe to retry and avoids duplicate Razorpay orders.
    - none, or `FAILED`: a new Razorpay order and a new `PENDING` payment are created.

    The amount sent to Razorpay is `Math.round(amount * 100)` (rupees to paise). If the Razorpay call fails, the API returns `503`.
3. **Client checkout.** The client opens the Razorpay widget with `razorpayOrderId` and `razorpayKeyId` (the public key id, never the secret). The backend has **no** "verify payment signature" endpoint for the client. The webhook is the only source of truth.
4. **Webhook** (`WebhooksService.handleRazorpayWebhook` in `src/webhooks/webhooks.service.ts`), see 5.3.

### 5.3 Webhook handling

`POST /webhooks/razorpay` is marked `@Public()` (Razorpay has no user token) and hidden from Swagger.

**Signature check.** Razorpay signs the exact bytes it sends. If we parsed the JSON and serialized it again, the bytes could change and the check would fail. So:

- `src/main.ts` creates the app with `{ rawBody: true }`, which keeps the original body on `req.rawBody`.
- The controller passes `req.rawBody` (as a UTF-8 string) and the `x-razorpay-signature` header to the service.
- `RazorpayService.verifyWebhookSignature` calls `Razorpay.validateWebhookSignature` with `RAZORPAY_WEBHOOK_SECRET`.
- Missing or wrong signature: `401`. This happens before any parsing or database work.

**Event filtering.** Only `payment.captured` and `payment.failed` are handled. Other events are logged and get `200`. A body with no payment entity is also logged and ignored.

**Dedup with `webhook_events`.** Razorpay can send the same event more than once. We build our own key: `eventId = "<event>:<razorpay payment id>"`, for example `payment.captured:pay_ABC123`. The column has a unique index (`UQ_webhook_events_event_id`).

- If a row with that key exists and `processed = true`, the webhook does nothing.
- Otherwise the payment is processed, and then a row is saved with `provider = RAZORPAY`, the event type, the full JSON payload, `processed = true`, and `processed_at`.
- The row is written **after** processing, and only on success. If processing throws, nothing is stored, we return an error, and Razorpay retries later.

**`payment.captured`** (`PaymentsService.markPaymentCaptured`, one transaction):

1. Find the payment by Razorpay order id. Not found: log and stop.
2. Already `SUCCESS`: stop. This is a second layer of idempotency.
3. Set the payment to `SUCCESS`, and store the Razorpay payment id and `paid_at`.
4. Set the order to `PAID`.
5. If the order is for a `SUBSCRIPTION`: suspend the user's other `ACTIVE` personal subscriptions, then set this one to `ACTIVE`.

**`payment.failed`** (`PaymentsService.markPaymentFailed`): set the payment to `FAILED` with Razorpay's `error_description`. The order and the subscription stay `PENDING`, so the user can call `POST /v1/payments/razorpay/create` again until the order expires. A later `payment.failed` never changes a payment that is already `FAILED` or `SUCCESS`.

### 5.4 Status values

| Entity | Values | Set by code today |
|---|---|---|
| Order | `PENDING`, `PAID`, `FAILED`, `FULFILLED` | `PENDING`, `PAID` only |
| Payment | `PENDING`, `SUCCESS`, `FAILED`, `REFUNDED` | `PENDING`, `SUCCESS`, `FAILED` |
| Subscription | `PENDING`, `ACTIVE`, `PAST_DUE`, `CANCELLED`, `EXPIRED`, `SUSPENDED` | `PENDING`, `ACTIVE`, `SUSPENDED` |
| Order resource type | `SUBSCRIPTION`, `AI_CREDITS` | Only `SUBSCRIPTION` is ever created or fulfilled |
| Payment provider | `RAZORPAY`, `STRIPE` | Only `RAZORPAY` |

### 5.5 Refunds

There is **no refund handling**: no endpoint, no `refund.*` webhook handling, and `REFUNDED` is never set. Refunds must be done by hand in the Razorpay dashboard, and the database will not show them.

## 6. Usage tracking and limits

### 6.1 The three methods

All in `src/commerce/usage/usage.service.ts`:

| Method | What it does |
|---|---|
| `consume(userId, featureKey, amount = 1, orgId = null)` | Checks the limit and adds to the counter, in one transaction with a row lock. Throws if the limit is hit. |
| `release(userId, featureKey, amount, orgId, chargedAt, manager?)` | Gives usage back when the work it paid for failed. Never goes below 0. Does nothing if the period has reset since `chargedAt`. Pass `manager` to run inside the caller's transaction. |
| `getFeatureValue(userId, featureKey, orgId = null)` | Reads a feature's value from the active plan without counting anything. Used for `enum` features. Never creates a free subscription. |

### 6.2 How `consume` works

```mermaid
sequenceDiagram
    participant Caller as Calling service
    participant U as UsageService
    participant DB as Postgres

    Caller->>U: consume userId, featureKey, 1, orgId
    U->>DB: find Feature by key
    U->>DB: BEGIN, find ACTIVE subscriptions for user or org
    alt no personal subscription
        U->>DB: create FREE subscription
    else no org subscription
        U-->>Caller: 422 organization has no active plan
    end
    U->>DB: find the plan feature for these plan versions
    U->>DB: SELECT usage row FOR UPDATE
    U->>U: create the row, or reset it if periodEnd has passed
    alt used + amount is over the limit
        U-->>Caller: 422 Usage limit exceeded
    else within the limit
        U->>DB: add amount to used, COMMIT
        U-->>Caller: used, limit, remaining, period
    end
```

Details:

- **Reset period.** `reset_period` is a number of days. When a new period starts, `period_start` = today and `period_end` = today + `reset_period` (UTC dates). If `reset_period` is `null`, `period_end` is `9999-12-31`, so it never resets.
- **Resets are lazy.** The next `consume` that sees `period_end <= now` sets `used = 0` and starts a new period. There is no reset job.
- **The counter belongs to the plan feature row.** When a user moves to a new plan or version, they start a new counter for the new plan feature.
- **Org pooling.** For an org, the usage row has `organization_id` set and `user_id` = the org subscription's purchaser. All members share it.

### 6.3 Who calls it

| Caller | File | Feature | Charge and refund |
|---|---|---|---|
| Generate assessment | `src/assessments/generation/generation-orchestrator.service.ts` `start` | `AI_ASSESSMENT` (personal or org) | Consumes 1 before creating jobs. Releases it if the transaction fails. |
| Retry a failed generation | same file, `retry` | `AI_ASSESSMENT` | Consumes 1 again. Releases it if requeueing fails. |
| Generation ends with failed sections | `src/assessments/generation/generation-aggregator.service.ts` `fail` | `AI_ASSESSMENT` | Releases the charge once (tracked by the `usage.refunded` flag in the job payload). |
| Question type check | `src/assessments/generation/generation-entitlement.service.ts` | `ALLOWED_QUESTION_TYPES` (read) | Not counted. Returns `403` naming the types the plan does not allow. |
| Older single-shot generate, used for roadmap node assessments | `src/assessments/assessments.service.ts` `generate`, called from `src/learning-roadmaps/learning-roadmap-assessment.service.ts` | `AI_ASSESSMENT` | Consumes 1. No release on failure. |
| Create a learning roadmap | `src/learning-roadmaps/learning-roadmaps.service.ts` `create` | `AI_LEARNING_ROADMAP` (personal only) | Consumes 1. No release. |
| Pick the LLM | `src/ai/ai-model-selector.service.ts` | `AI_MODEL` (read, personal only) | Not counted. |
| Profile (`GET /v1/auth/me`) | `src/identity/auth/auth.service.ts` | Current plan and allowed question types | Read only (`getCurrentPlan`, `getAllowedQuestionTypes`). |

See [Assessment generation](assessment-generation.md) and [Learning roadmaps](learning-roadmaps.md) for the rest of those flows.

### 6.4 What happens when a limit is hit

| Situation | HTTP status | Message |
|---|---|---|
| Limit reached | `422` | `Usage limit exceeded for feature AI_ASSESSMENT: 2/2 used` |
| Org request, org has no `ACTIVE` subscription | `422` | `This organization has no active plan` |
| Feature is `enum`, not `limit` | `422` | `Feature X is not a usage-limited feature` |
| Active plan does not include the feature | `404` | `Feature X is not included in the user's active plan` |
| Feature key missing from the `features` table | `404` | `Feature X not found` |
| Plan does not allow a question type | `403` | `Your plan does not include ... questions; it allows: ...` |

## 7. Reading your subscription and usage

- `GET /v1/subscriptions`: the current `ACTIVE` subscription, with plan, version and features. With `x-organization-id`, the org's subscription (needs the org permission `org_billing:read`). `404` if there is none.
- `GET /v1/usage/subscriptions/:subscriptionId`: every feature of that subscription's plan version, with `used`, `remaining`, and the period. A counter whose period has ended shows `used = 0`. Same org rules as above.
- `GET /v1/plans`: public (no login). Active and displayed plans, each with only its **current** version and features. Paginated, and filterable by `name` and `audience`.

## 8. Admin endpoints

All under `/v1/admin`, in `src/commerce/admin.controller.ts`, served by `CommerceAdminService`. Each needs a platform permission (see [RBAC](rbac.md)).

| Method and path | Permission | What it does |
|---|---|---|
| `GET /plans` | `plans:read` | All plans, including hidden and inactive ones. Filters: `isActive`, `isDisplayed`, `name`, `audience`. |
| `GET /plans/:planId` | `plans:read` | One plan with every version (newest first) and their features. |
| `POST /plans` | `plans:create` | Create a plan and its version 1 (no features yet). |
| `PATCH /plans/:planId` | `plans:update` | Edit name, description, `isActive`, `isDisplayed`, `sortOrder`. Built-in plans cannot be renamed or deactivated. |
| `POST /plans/:planId/versions` | `plans:create` | Add a new version and make it current. |
| `PUT /plans/:planId/versions/:versionNo` | `plans:update` | Edit a version in place. `isCurrent: true` makes it the current one. |
| `POST /plans/:planId/versions/:versionNumber/features` | `plans:create` | Attach a feature to a version. |
| `PUT /plans/:planId/versions/:versionNumber/features/:featureId` | `plans:update` | Edit a plan feature. `:featureId` is the `plan_features` row id. |
| `DELETE /plans/:planId/versions/:versionNumber/features/:featureId` | `plans:delete` | Soft-delete a plan feature. Subscribers lose it at once. Usage history is kept. |
| `GET /features` | `features:read` | List all features (for the "add feature" picker). |
| `GET /payments` | `payments:read` | All payments, with the payer and the plan bought. Filters: status, provider, user, date range, search. |
| `GET /subscriptions` | `subscriptions:read` | All subscriptions. Filters: status, user, plan, `scope` (personal or organization), `organizationUid`, search. |
| `GET /usage` | `usage:read` | All usage rows. Filters: user, `featureKey`, `current`, `scope`, `organizationUid`, search. |

## 9. Business rules and edge cases

1. Every verified user should have one `ACTIVE` personal subscription. If they have none when they spend usage, a `FREE` one is created on the spot.
2. Organizations never fall back to a member's personal plan. No `ACTIVE` org subscription means metered AI features are refused with `422`.
3. Only `individual` plans can be bought. Organizations stay on `ORG_TRIAL` because there is no org checkout.
4. Checkout blocks buying a plan you already have `ACTIVE`. It does **not** block a second checkout while one is still `PENDING`.
5. Paying for a new personal plan suspends all other `ACTIVE` personal plans, `FREE` included. Org subscriptions are not touched.
6. An order expires 30 minutes after checkout (`CHECKOUT_ORDER_TTL_MINUTES`). After that, `POST /v1/payments/razorpay/create` returns `422`, and the user must check out again.
7. A `payment.captured` webhook is still honored if it arrives after the order expired. Expiry only blocks starting a payment.
8. Only the webhook updates our database. If the webhook never arrives, the subscription stays `PENDING` even though the user paid.
9. A limit value that is not a number (for example `unlimited`) means unlimited.
10. Usage resets lazily on the next `consume`, using UTC dates.
11. `release` only refunds into the same period. A charge made before the current `period_start` is not given back.
12. A plan version can be bought until its `effective_until`, even if it is no longer the current one.
13. Editing a version's price with `PUT .../versions/:versionNo` changes it in place, so later checkouts of that version charge the new price.

## 10. Endpoints summary

| Method | Path | Auth |
|---|---|---|
| GET | `/v1/plans` | Public |
| GET | `/v1/subscriptions` | User (org: `org_billing:read`) |
| POST | `/v1/subscriptions/checkout` | User |
| POST | `/v1/payments/razorpay/create` | User |
| GET | `/v1/usage/subscriptions/:subscriptionId` | User (org: `org_billing:read`) |
| POST | `/webhooks/razorpay` | Public, Razorpay signature |
| various | `/v1/admin/plans...`, `/v1/admin/features`, `/v1/admin/payments`, `/v1/admin/subscriptions`, `/v1/admin/usage` | Platform permissions |

Full request and response details are in [API Design v1](../api-design/v1.md).

## 11. Where the code lives

| Path | What |
|---|---|
| `src/commerce/commerce.module.ts` | Module wiring and exports |
| `src/commerce/plans/` | Public plan list, `plan.constants.ts`, `allowed-question-types.ts`, plan DTOs |
| `src/commerce/subscriptions/subscriptions.service.ts` | Checkout, free and trial subscriptions, current plan lookups |
| `src/commerce/usage/usage.service.ts` | `consume`, `release`, `getFeatureValue`, usage report |
| `src/commerce/admin.controller.ts`, `admin.service.ts` | Admin catalog and reporting |
| `src/commerce/serializers/commerce.serializer.ts` | Response shapes |
| `src/commerce/payments/payments.service.ts` | Razorpay payment creation, captured and failed handling |
| `src/commerce/payments/razorpay/razorpay.service.ts` | Razorpay SDK client and signature check |
| `src/webhooks/` | Webhook controller and dedup logic |
| `src/common/entities/` | `plan`, `plan-version`, `plan-version-feature`, `feature`, `subscription`, `user-feature-usage`, `order`, `payment`, `webhook-event` |
| `src/common/enums/commerce.enums.ts` | All commerce enums |
| `src/seeders/feature.seeder.ts`, `plan.seeder.ts` | Seed data |

Environment variables: `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET`, and `CHECKOUT_ORDER_TTL_MINUTES` (optional).

## 12. Known gaps and TODOs

1. **No expiry.** Nothing moves a subscription to `EXPIRED`, and lookups ignore `expires_at`. The 7-day org trial and monthly plans never actually end.
2. **No renewal, cancel, or refund.** `auto_renew`, `cancelled_at`, `CANCELLED`, `PAST_DUE`, `REFUNDED`, and `FULFILLED` are unused.
3. **No org checkout.** Organization plans cannot be bought.
4. **`AI_CREDITS` orders have no fulfillment.** Only `SUBSCRIPTION` is handled in `markPaymentCaptured`, with a `TODO` to move to an event-driven handler per resource type.
5. **Failed payments do not fail the order.** The order stays `PENDING` until it expires. Stale `PENDING` orders and subscriptions are never cleaned up.
6. **The seeded `FREE` and `ORG_TRIAL` plans do not include `AI_LEARNING_ROADMAP`**, so creating a roadmap fails with `404` until an admin adds that feature to the plan.
7. **No unique index on `user_feature_usage`** for (user or org, plan feature). Two first-time `consume` calls at the same moment can create two counter rows.
8. **`AI_MODEL` is read from the personal plan only**, even on org requests.
9. **The webhook does not check the amount.** We trust that the captured payment matches the order, because the Razorpay order was created with our amount.
10. **Zero-price plans through checkout** would send a 0 amount order to Razorpay, which Razorpay is expected to reject, giving the user `503`.
