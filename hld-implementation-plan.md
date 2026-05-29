# TestFlow — High Level Design: Implementation Plan

---

# 1. Project Overview

## 1.1 Purpose

TestFlow is a production-grade online assessment platform enabling instructors to create AI-assisted tests, manage participants, collect submissions, and evaluate results — including LLM-powered question generation and subjective answer scoring.

## 1.2 Goals

- Deliver a scalable, secure assessment platform handling concurrent test-takers
- Reduce test creation friction via LLM-assisted question generation from a topic/description
- Support MCQ (auto-scored), Coding (test-case-scored), and Subjective (LLM-scored) question types
- Enforce strict role-based access control (RBAC) across all operations
- Enable analytics, leaderboards, and usage reporting for platform operators

## 1.3 Non-Goals (Phase 1)

- Real-time collaborative editing of tests
- Video proctoring / screen recording
- Payment processing (schema deferred; integration is Phase 1 scope but requires schema definition first)
- Mobile-native apps (web-first)

## 1.4 Tech Stack (Inferred/Confirmed)

| Layer | Choice | Rationale |
|---|---|---|
| Backend framework | NestJS (TypeScript) | Modular monolith, decorator-based DI |
| ORM | TypeORM | NestJS-native, migration support |
| Database | PostgreSQL 15+ | JSONB support, row-level locking |
| Auth | JWT (access + refresh tokens) | Stateless, scalable |
| Cache / Session | Redis | Token blacklist, rate limiting, leaderboard cache |
| LLM integration | OpenAI / Anthropic (configurable) | Question generation + subjective eval |
| Queue | BullMQ (Redis-backed) | Async LLM jobs, code execution |
| Code execution | Isolated sandbox (Judge0 or self-hosted) | CODING question test-case evaluation |
| API layer | REST, versioned at `/api/v1` | Stable contract; GraphQL deferred |
| Frontend | Next.js (React, TypeScript) | SSR for test listings, CSR for test-taking |
| Infra | Docker + docker-compose (dev), ECS/K8s (prod) | Environment parity |
| CI/CD | GitHub Actions | Lint → test → build → deploy |
| Observability | OpenTelemetry + Datadog/Grafana | Traces, metrics, logs |

---

# 2. Architecture Summary

## 2.1 Current Architecture (v2.1.0)

```
Client
  │
  ├─ AuthController
  ├─ TestController
  └─ SubmissionController
        │
  ├─ AuthService
  ├─ TestService ──────► LLMModule (question generation)
  └─ SubmissionService ─► EvaluationLogic
        │
  PostgreSQL (single DB)
```

## 2.2 Target Architecture (Phase 1 End State)

```
Client (Next.js)
  │  HTTPS
  ▼
API Gateway / Reverse Proxy (Nginx)
  │
  ▼
NestJS Modular Monolith (/api/v1)
  ├── AuthModule          (JWT, OAuth, OTP, Password reset)
  ├── UsersModule         (CRUD, soft-delete, activation)
  ├── RBACModule          (Roles, Permissions, Resource keys)
  ├── TestsModule         (Create/Read/Update/Delete tests)
  ├── QuestionsModule     (CRUD, LLM generation)
  ├── ParticipantsModule  (Enrollment, state machine)
  ├── SubmissionsModule   (Answer submission, scoring)
  ├── EvaluationModule    (MCQ auto, Coding sandbox, LLM subjective)
  ├── AnalyticsModule     (Stats, leaderboard, usage)
  └── LLMModule           (Provider abstraction: OpenAI/Anthropic)
        │                       │
        ▼                       ▼
   PostgreSQL 15+           BullMQ (Redis)
   (primary DB)              ├── llm-generation queue
                             ├── llm-evaluation queue
                             └── code-execution queue
                                      │
                                      ▼
                               Judge0 / Code Sandbox
```

## 2.3 Key Architectural Decisions

**Modular Monolith over Microservices**: Easier to develop, test, and deploy at current team size. Module boundaries are enforced via NestJS module encapsulation — migration to microservices is possible per-module later.

**Async LLM calls via BullMQ**: LLM calls are non-deterministic in latency (1–30s). All LLM operations (question generation, subjective scoring) are queued. REST endpoints return a job ID; clients poll or use SSE for completion.

**PostgreSQL JSONB for options/test_cases**: Avoids premature normalization for question-type-specific payloads. Indexed via GIN for filtering.

**Redis for token blacklist**: Logout invalidates refresh tokens by inserting into a Redis set with TTL matching token expiry. Access tokens remain valid until expiry (short-lived, ~15 min).

**Optimistic concurrency for participant slot reservation**: `UPDATE tests SET current_participants = current_participants + 1 WHERE id = $1 AND current_participants < max_participants` — single atomic statement, no application-level lock needed.

---

# 3. Feature and Module Breakdown

## 3.1 Module Registry

| Module | Purpose | Tables Touched | APIs | Phase |
|---|---|---|---|---|
| AuthModule | Authentication, token lifecycle, OAuth, OTP | users | /auth/* | 1 |
| UsersModule | User CRUD, profile, activation | users, users_roles | /users/* | 1 |
| RBACModule | Role/permission management, access checks | roles, users_roles, resources, permissions | /roles/*, /permissions/* | 1 |
| TestsModule | Test lifecycle management | tests, tests_questions, tests_participants | /tests/* | 1 |
| QuestionsModule | Question CRUD + LLM generation | questions, tests_questions | /questions/* | 1 |
| ParticipantsModule | Enrollment, state transitions | tests_participants | /tests/{id}/participants | 1 |
| SubmissionsModule | Answer collection, attempt tracking | submissions, tests_questions | /submissions/* | 1 |
| EvaluationModule | Scoring: MCQ/Coding/Subjective | submissions, tests_questions, questions | Internal + queue workers | 1 |
| LLMModule | Provider abstraction, prompt management | — (external API) | Internal service | 1 |
| AnalyticsModule | Stats, leaderboard, usage | All (read-only aggregations) | /analytics/*, /tests/{id}/stats | 1 |
| PaymentsModule | Subscription plans, billing | TBD (schema missing) | TBD | 1 (schema first) |

## 3.2 Module Detail: AuthModule

**Purpose**: Handle all authentication flows — local credentials, OAuth (Google/GitHub), OTP-based login, JWT issuance/refresh, and password reset.

**Components impacted**: `AuthController`, `AuthService`, `JwtStrategy`, `OAuthStrategy`, `OtpService`, `MailService`

**APIs involved**: All `/auth/*` endpoints

**DB tables**: `users` (oauth_provider, oauth_id, is_verified fields)

**Business rules**:
- Access token TTL: 15 minutes; Refresh token TTL: 7 days
- OTP TTL: 10 minutes, stored in Redis (key: `otp:{email}`, value: hashed OTP)
- Refresh tokens stored in Redis (key: `refresh:{userId}:{tokenId}`); logout deletes key
- OAuth: upsert user on callback — if `(oauth_provider, oauth_id)` exists, login; else create user
- Password reset tokens: short-lived (1 hour), stored in Redis
- Brute force protection: 5 failed login attempts → 15-minute lockout via Redis counter

**State machine**: Token states — ISSUED → ACTIVE → EXPIRED | REVOKED

**Dependencies**: MailService (SMTP), Redis, users table

**Risks/Edge cases**:
- OAuth account linking (same email, different provider) — determine policy: auto-link by email or reject
- Concurrent refresh token usage (token rotation race) — implement refresh token family tracking

**Implementation order**: 1. Local auth (login/register) → 2. JWT refresh/logout → 3. OTP → 4. OAuth → 5. Password reset

## 3.3 Module Detail: RBACModule

**Purpose**: Manage roles, resources, and permissions. Enforce access control on every protected endpoint.

**Components**: `RBACController`, `RBACService`, `RBACGuard`, `PermissionsDecorator`

**Business rules**:
- Resource keys are slug-style strings: `tests:create`, `questions:read`, `users:delete`
- Default roles: `ADMIN`, `INSTRUCTOR`, `PARTICIPANT` — seeded via migration
- `PUT /roles/{roleKey}/permissions` replaces entire permission set (idempotent)
- `/access-check` endpoint for frontend pre-flight permission checks
- Guard intercepts every request, reads `@RequirePermission('resource:action')` decorator, queries permissions cache

**DB tables**: `roles`, `resources`, `permissions`, `users_roles`

**Performance**: Cache role-permission map in Redis (key: `rbac:{roleId}`). Invalidate on `PUT /roles/{roleKey}/permissions`. Guard reads from cache with fallback to DB.

**Dependencies**: Redis (cache), AuthModule (userId from JWT)

**Risks**: Circular permission dependencies are not possible in this flat model. Risk is stale cache after permission update — use pub/sub invalidation or short TTL (60s).

## 3.4 Module Detail: TestsModule

**Purpose**: Full lifecycle management of tests — creation (including LLM-generated questions), configuration, listing, and deletion.

**Components**: `TestController`, `TestService`, `TestRepository`, `TestQueryBuilder`

**Business rules**:
- `created_by` set to authenticated user's ID on creation
- `current_participants` must never exceed `max_participants` (enforced at DB level, not application level)
- Test status is a derived field — computed from `start_time`/`end_time` vs `NOW()` — never stored
- Soft delete not in schema for tests (no `deleted_at`) — confirm with team; hard delete with cascade constraints
- Test editing allowed only before `start_time` (business rule to enforce)
- LLM-assisted creation: POST /tests with `description` and `question_config` triggers async job; returns `jobId`

**DB tables**: `tests`, `tests_questions`, `tests_participants`

**Dependencies**: QuestionsModule (for question attachment), LLMModule (async generation), ParticipantsModule

**Risks**: Long-running LLM job — client must handle pending state; job failure leaves test in draft state.

## 3.5 Module Detail: QuestionsModule

**Purpose**: Standalone question bank management plus question generation via LLM.

**Components**: `QuestionController`, `QuestionService`, `QuestionRepository`, `LLMGenerationService`

**Business rules**:
- Questions are owned by creator (`created_by`)
- MCQ: `options` jsonb required, `test_cases` null
- CODING: `test_cases` jsonb required (array of `{input, expected_output, is_hidden}`)
- SUBJECTIVE: both `options` and `test_cases` null
- `unused` filter: questions not in any `tests_questions` row
- LLM generation: POST /questions with `{generate: true, count: N, type: MCQ, topic: "..."}` → async job

**DB tables**: `questions`, `tests_questions`

**Indexes**: GIN index on `options` and `test_cases` jsonb columns; B-tree on `created_by`, `type`

## 3.6 Module Detail: ParticipantsModule

**Purpose**: Manage test enrollment and state transitions for participants.

**Components**: `ParticipantController`, `ParticipantService`, `ParticipantRepository`

**Business rules**:
- State machine: `NOT_STARTED → IN_PROGRESS → COMPLETED` (no reversals)
- A user can only have one active (non-deleted) `tests_participants` row per test
- `IN_PROGRESS` entry is created when user starts the test (not when they enroll — clarify with team whether enrollment exists as a pre-step)
- `current_participants` incremented atomically when row is inserted
- `completed_at` set when status transitions to `COMPLETED`
- After `end_time`, system auto-completes `IN_PROGRESS` participants (background job)

**Concurrency**:
```sql
UPDATE tests
SET current_participants = current_participants + 1
WHERE id = $1 AND current_participants < max_participants
RETURNING id;
```
If no row returned → test is full → reject with 409.

**Dependencies**: TestsModule (capacity check), SubmissionsModule (visibility gate)

## 3.7 Module Detail: SubmissionsModule

**Purpose**: Collect answers, track attempts, trigger evaluation, and expose results.

**Components**: `SubmissionController`, `SubmissionService`, `SubmissionRepository`, `EvaluationOrchestrator`

**Business rules**:
- Submission only accepted if participant status = `IN_PROGRESS`
- Submission rejected if `test.end_time < NOW()`
- Multiple submissions per question allowed; `attempt_no` auto-incremented per `(participant_id, test_question_id)`
- Latest attempt wins for scoring (highest `attempt_no`)
- Submission results only visible after participant status = `COMPLETED`
- `evaluated_result` jsonb stores evaluation detail (MCQ: correct/incorrect + correct_option; CODING: per-test-case results; SUBJECTIVE: LLM rationale + score)

**DB tables**: `submissions`, `tests_questions`, `tests_participants`

**Concurrency**: `attempt_no` computed as `SELECT MAX(attempt_no) + 1 FROM submissions WHERE participant_id = $1 AND test_question_id = $2` — wrap in serializable transaction or use `SELECT ... FOR UPDATE` on the participant row.

## 3.8 Module Detail: EvaluationModule

**Purpose**: Score submissions based on question type. Operates as internal service + queue worker.

**MCQ Evaluation**:
```typescript
// options jsonb structure: [{text: string, isCorrect: boolean}]
const correct = question.options.find(o => o.isCorrect);
const isCorrect = submission.answer === correct.text; // or index-based
const score = isCorrect ? marks : -negativeMarks;
```

**CODING Evaluation**:
- Dispatch to Judge0 (or equivalent sandbox) with code + test cases
- Store per-test-case result in `evaluated_result` jsonb
- Score = (passing_test_cases / total_test_cases) × marks
- Hidden test cases not exposed to participant

**SUBJECTIVE Evaluation**:
- LLM prompt: system context (question text, expected criteria if any) + participant answer
- Parse structured response: `{score: number, max_score: number, rationale: string, confidence: float}`
- Store in `evaluated_result` jsonb
- Human review flag for low-confidence evaluations

**Queue design (BullMQ)**:
- `evaluation:mcq` — processed synchronously in worker (fast)
- `evaluation:coding` — Judge0 API call, retry on failure, 3 retries with backoff
- `evaluation:subjective` — LLM call, retry on 5xx, dead-letter queue for manual review

## 3.9 Module Detail: LLMModule

**Purpose**: Abstract LLM provider, manage prompts, handle retries, and parse structured outputs.

**Design**:
```typescript
interface LLMProvider {
  generateQuestions(params: GenerationParams): Promise<Question[]>;
  evaluateSubjective(params: EvalParams): Promise<EvalResult>;
}
// Implementations: OpenAIProvider, AnthropicProvider
// Selected via config: LLM_PROVIDER=openai|anthropic
```

**Prompt templates** stored in `src/llm/prompts/` as versioned `.hbs` (Handlebars) files — allows non-code updates to prompts.

**Output parsing**: Require JSON-mode responses. Validate against Zod schema before persisting. Retry with clarification prompt on parse failure (max 2 retries).

**Rate limiting**: Implement token bucket per provider. Respect provider rate limits (TPM/RPM). Log all LLM calls with token counts for cost tracking.

**Observability**: Log `{provider, model, prompt_tokens, completion_tokens, latency_ms, job_id}` per call.

## 3.10 Module Detail: AnalyticsModule

**Purpose**: Aggregated read-only reporting — test stats, user stats, leaderboard, platform usage.

**Key queries**:

Leaderboard:
```sql
SELECT tp.user_id, u.full_name,
  SUM(s.score) as total_score,
  MIN(tp.completed_at) as completion_time
FROM tests_participants tp
JOIN submissions s ON s.participant_id = tp.id
JOIN users u ON u.id = tp.user_id
WHERE tp.test_id = $1
  AND tp.status = 'COMPLETED'
  AND s.attempt_no = (
    SELECT MAX(s2.attempt_no) FROM submissions s2
    WHERE s2.participant_id = s.participant_id
    AND s2.test_question_id = s.test_question_id
  )
GROUP BY tp.user_id, u.full_name
ORDER BY total_score DESC, completion_time ASC;
```

**Caching**: Leaderboard cached in Redis (key: `leaderboard:{testId}`, TTL: 60s). Invalidated on new submission scoring completion.

**Performance**: Test stats (participant count, avg score, completion rate) precomputed on test completion event and stored in a `test_stats` materialized view — refreshed via trigger or scheduled job.

---

# 4. Dependency Mapping

## 4.1 Module Dependency Graph

```
AuthModule
    └─► UsersModule (user lookup, creation)
    └─► Redis (token storage)
    └─► MailService (OTP, password reset)

UsersModule
    └─► RBACModule (role assignment)

RBACModule
    └─► Redis (permission cache)

TestsModule
    └─► UsersModule (ownership validation)
    └─► QuestionsModule (attach questions)
    └─► LLMModule (question generation)
    └─► ParticipantsModule (capacity enforcement)

QuestionsModule
    └─► LLMModule (generation)
    └─► TestsModule (unused filter)

ParticipantsModule
    └─► TestsModule (test existence, capacity)
    └─► SubmissionsModule (visibility gate)

SubmissionsModule
    └─► ParticipantsModule (status check)
    └─► TestsModule (timing check)
    └─► EvaluationModule (trigger scoring)

EvaluationModule
    └─► LLMModule (subjective scoring)
    └─► Judge0 (coding evaluation)
    └─► BullMQ (async job dispatch)
    └─► SubmissionsModule (write back scores)

AnalyticsModule
    └─► All modules (read-only queries)
    └─► Redis (cache)

PaymentsModule
    └─► UsersModule (subscription state)
    └─► External payment provider (Stripe)
```

## 4.2 External Dependencies

| Dependency | Purpose | Failure mode | Mitigation |
|---|---|---|---|
| OpenAI/Anthropic API | Question gen + eval | Rate limit, timeout, outage | Circuit breaker, retry with backoff, fallback to manual |
| Judge0 | Code execution | Sandbox timeout, outage | BullMQ retry, dead-letter queue, per-language timeout config |
| SMTP (SendGrid/SES) | OTP, password reset | Delivery failure | Retry queue, fallback provider |
| Redis | Cache, queues, blacklist | Cluster failure | Sentinel/Cluster mode; auth still works via DB fallback for permissions |
| PostgreSQL | Primary data store | Connection pool exhaustion, replication lag | PgBouncer, read replica for analytics, connection limit config |
| Stripe | Payments | Webhook failure | Idempotency keys, webhook replay |

---

# 5. Database Design and Migration Plan

## 5.1 Migration Strategy

Use TypeORM migrations (not `synchronize: true` in production). All schema changes go through numbered migration files committed to version control.

Migration naming: `{timestamp}-{description}.ts` — e.g., `1700000001-create-users-table.ts`

Seed migrations: separate `seeds/` directory run via `npm run seed` — not bundled with schema migrations.

## 5.2 Index Recommendations

```sql
-- users table
CREATE UNIQUE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX idx_users_username ON users(username) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX idx_users_phone ON users(phone_number) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX idx_users_oauth ON users(oauth_provider, oauth_id) WHERE oauth_id IS NOT NULL;
CREATE INDEX idx_users_is_active ON users(is_active) WHERE deleted_at IS NULL;

-- tests table
CREATE INDEX idx_tests_created_by ON tests(created_by);
CREATE INDEX idx_tests_start_time ON tests(start_time);
CREATE INDEX idx_tests_end_time ON tests(end_time);
CREATE INDEX idx_tests_time_range ON tests(start_time, end_time);

-- questions table
CREATE INDEX idx_questions_created_by ON questions(created_by);
CREATE INDEX idx_questions_type ON questions(type);
CREATE INDEX idx_questions_options_gin ON questions USING GIN(options);
CREATE INDEX idx_questions_test_cases_gin ON questions USING GIN(test_cases);

-- tests_questions table
CREATE INDEX idx_tests_questions_test_id ON tests_questions(test_id);
CREATE INDEX idx_tests_questions_question_id ON tests_questions(question_id);
CREATE UNIQUE INDEX idx_tests_questions_unique ON tests_questions(test_id, question_id);

-- tests_participants table
CREATE INDEX idx_tp_test_id ON tests_participants(test_id);
CREATE INDEX idx_tp_user_id ON tests_participants(user_id);
CREATE INDEX idx_tp_status ON tests_participants(status) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX idx_tp_user_test_active ON tests_participants(user_id, test_id)
  WHERE deleted_at IS NULL AND status != 'COMPLETED';

-- submissions table
CREATE INDEX idx_sub_participant_id ON submissions(participant_id);
CREATE INDEX idx_sub_test_question_id ON submissions(test_question_id);
CREATE INDEX idx_sub_submitted_at ON submissions(submitted_at);
CREATE INDEX idx_sub_score ON submissions(score) WHERE score IS NOT NULL;
CREATE INDEX idx_sub_latest ON submissions(participant_id, test_question_id, attempt_no DESC);

-- permissions table
CREATE INDEX idx_permissions_role_id ON permissions(role_id);
CREATE UNIQUE INDEX idx_permissions_role_resource ON permissions(role_id, resource_id) WHERE deleted_at IS NULL;
```

## 5.3 Missing Schema: Payments

No payments schema exists. Required additions before PaymentsModule development:

```sql
CREATE TABLE subscription_plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR NOT NULL,
  price_cents INTEGER NOT NULL,
  billing_interval VARCHAR NOT NULL, -- 'MONTHLY', 'ANNUAL'
  max_tests INTEGER,
  max_questions_per_test INTEGER,
  llm_generation_credits INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE user_subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  plan_id UUID NOT NULL REFERENCES subscription_plans(id),
  stripe_subscription_id VARCHAR UNIQUE,
  status VARCHAR NOT NULL, -- 'ACTIVE', 'CANCELLED', 'PAST_DUE'
  current_period_start TIMESTAMPTZ,
  current_period_end TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

## 5.4 Migration Order

1. `create-users` → `create-roles` → `create-users-roles`
2. `create-resources` → `create-permissions`
3. `create-tests` → `create-questions`
4. `create-tests-questions` → `create-tests-participants`
5. `create-submissions`
6. `seed-default-roles` → `seed-default-resources` → `seed-default-permissions`
7. `create-subscription-plans` → `create-user-subscriptions` (Phase 1, after schema approval)

## 5.5 Soft Delete Implementation

Entities with soft delete (`users`, `roles`, `resources`, `permissions`, `tests_participants`) must use TypeORM `@DeleteDateColumn()`. All repository queries must include `WHERE deleted_at IS NULL` — enforce this via a `BaseRepository` class that wraps TypeORM's `find` methods.

```typescript
abstract class SoftDeletableRepository<T> {
  protected addSoftDeleteFilter(qb: SelectQueryBuilder<T>): void {
    qb.andWhere(`${this.alias}.deletedAt IS NULL`);
  }
}
```

---

# 6. API Implementation Plan

## 6.1 API Contract Standards

- Base path: `/api/v1`
- All responses follow envelope: `{success, data, meta, error}`
- `meta` only present on paginated responses: `{page, limit, total, totalPages}`
- Error envelope: `{success: false, data: null, error: {code, message, details}}`
- HTTP status codes: 200 (OK), 201 (Created), 204 (No Content), 400 (Validation), 401 (Unauth), 403 (Forbidden), 404 (Not Found), 409 (Conflict), 422 (Business Rule Violation), 429 (Rate Limited), 500 (Internal)
- Use 422 (not 400) for business rule violations (e.g., test full, double enrollment)

## 6.2 Validation Layer

All incoming DTOs validated via `class-validator` + `class-transformer` in NestJS `ValidationPipe` (global, with `whitelist: true, forbidNonWhitelisted: true, transform: true`).

Custom validators:
- `@IsQuestionType()` — validates MCQ requires options, CODING requires test_cases
- `@IsTimeRange()` — validates end_time > start_time
- `@IsUUID()` — all ID params

## 6.3 Key API Design Decisions

**Test status filter implementation** (applied at query level):
```typescript
if (filter.status === 'upcoming')  qb.andWhere('test.start_time > NOW()');
if (filter.status === 'ongoing')   qb.andWhere('test.start_time <= NOW() AND test.end_time >= NOW()');
if (filter.status === 'completed') qb.andWhere('test.end_time < NOW()');
```

**LLM-backed async endpoints** (POST /questions with generate:true, POST /tests with LLM questions):
- Return immediately with `{jobId, status: 'PENDING'}`
- Client polls `GET /jobs/{jobId}` or subscribes to SSE `GET /jobs/{jobId}/stream`
- Job states: `PENDING → PROCESSING → COMPLETED | FAILED`
- Store job state in Redis (key: `job:{jobId}`, TTL: 24h)

**Pagination**: All list endpoints support `?page=1&limit=20` (default limit: 20, max: 100). Use keyset pagination for leaderboard (offset pagination performance degrades at scale).

## 6.4 Rate Limiting

```
POST /auth/login           → 10 req/min per IP
POST /auth/otp             → 3 req/10min per email
POST /auth/password/forgot → 3 req/hour per email
POST /submissions          → 60 req/min per user (prevents spam submission)
LLM generation endpoints   → 10 req/hour per user (cost control)
```

Implemented via `@nestjs/throttler` + Redis store.

## 6.5 Authorization Matrix

| Endpoint | ADMIN | INSTRUCTOR | PARTICIPANT |
|---|---|---|---|
| POST /tests | ✓ | ✓ | ✗ |
| PATCH /tests/{id} | ✓ | owner only | ✗ |
| DELETE /tests/{id} | ✓ | owner only | ✗ |
| POST /questions | ✓ | ✓ | ✗ |
| POST /tests/{id}/participants | ✓ | ✓ | self-enroll |
| GET /tests/{id}/submissions | ✓ | owner only | own only |
| GET /analytics/usage | ✓ | ✗ | ✗ |
| GET /roles | ✓ | ✗ | ✗ |
| PUT /roles/{key}/permissions | ✓ | ✗ | ✗ |
| PATCH /users/{username}/activate | ✓ | ✗ | ✗ |

---

# 7. Backend Implementation Plan

## 7.1 Project Structure

```
src/
├── common/
│   ├── decorators/         # @CurrentUser, @RequirePermission, @Public
│   ├── filters/            # GlobalExceptionFilter
│   ├── guards/             # JwtAuthGuard, RBACGuard, ThrottlerGuard
│   ├── interceptors/       # ResponseEnvelopeInterceptor, LoggingInterceptor
│   ├── pipes/              # ValidationPipe config
│   ├── dto/                # PaginationDto, ResponseDto<T>
│   └── utils/              # date utils, cursor encoder
├── config/                 # ConfigModule, typed config schemas (Zod)
├── database/
│   ├── migrations/
│   ├── seeds/
│   └── entities/           # shared base entity
├── modules/
│   ├── auth/
│   ├── users/
│   ├── rbac/
│   ├── tests/
│   ├── questions/
│   ├── participants/
│   ├── submissions/
│   ├── evaluation/
│   ├── llm/
│   └── analytics/
├── queues/
│   ├── processors/         # BullMQ workers
│   └── producers/          # Job dispatchers
└── main.ts
```

## 7.2 BaseEntity

```typescript
@Entity()
export abstract class BaseEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}

export abstract class SoftDeletableEntity extends BaseEntity {
  @DeleteDateColumn()
  deletedAt: Date | null;
}
```

## 7.3 Repository Pattern

Each module has its own repository class wrapping TypeORM's `Repository<T>`. Never inject `DataSource` directly in services. All complex queries use `QueryBuilder`. Never use raw SQL except in migrations and analytics aggregations.

```typescript
@Injectable()
export class TestRepository {
  constructor(
    @InjectRepository(Test)
    private readonly repo: Repository<Test>,
  ) {}

  async findWithFilters(filters: TestFilters): Promise<[Test[], number]> {
    const qb = this.repo.createQueryBuilder('test');
    // Apply filters
    return qb.getManyAndCount();
  }
}
```

## 7.4 Transaction Management

Use TypeORM's `DataSource.transaction()` for multi-step writes. Identify all operations requiring transactions:

| Operation | Transaction scope |
|---|---|
| User register | Insert user + assign default role |
| Enroll participant | Increment `current_participants` + insert `tests_participants` row |
| Submit answer | Insert submission + increment `attempt_no` (advisory lock or serializable) |
| Complete test | Update `tests_participants.status` + trigger evaluation for all questions |
| PUT /roles/{key}/permissions | Delete existing permissions + insert new set |

## 7.5 Configuration Management

Use `@nestjs/config` with Zod validation schema:

```typescript
const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_ACCESS_SECRET: z.string().min(32),
  JWT_REFRESH_SECRET: z.string().min(32),
  JWT_ACCESS_TTL: z.string().default('900'),   // seconds
  JWT_REFRESH_TTL: z.string().default('604800'),
  LLM_PROVIDER: z.enum(['openai', 'anthropic']).default('openai'),
  OPENAI_API_KEY: z.string().optional(),
  ANTHROPIC_API_KEY: z.string().optional(),
  JUDGE0_URL: z.string().url().optional(),
  SMTP_HOST: z.string(),
  SMTP_PORT: z.coerce.number(),
  STRIPE_SECRET_KEY: z.string().optional(),
  STRIPE_WEBHOOK_SECRET: z.string().optional(),
});
```

All secrets injected via environment variables — never committed. Use AWS Secrets Manager or Vault in production.

## 7.6 Error Handling

Global exception filter maps all errors to the response envelope format:

```typescript
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    // Map TypeORM errors (23505=unique violation, 23503=FK violation)
    // Map HttpException
    // Map custom BusinessRuleException (→ 422)
    // Log unexpected errors to observability platform
  }
}
```

Custom exception classes:
- `BusinessRuleException` (422) — test full, double enrollment, invalid state transition
- `ResourceNotFoundException` (404)
- `ConflictException` (409) — unique constraint human-readable message
- `EvaluationException` — internal, triggers dead-letter queue

## 7.7 BullMQ Queue Setup

```typescript
export const QUEUES = {
  LLM_GENERATION: 'llm-generation',
  LLM_EVALUATION: 'llm-evaluation',
  CODE_EXECUTION:  'code-execution',
  AUTO_COMPLETE:   'auto-complete',
} as const;

const defaultJobOptions: JobsOptions = {
  attempts: 3,
  backoff: { type: 'exponential', delay: 2000 },
  removeOnComplete: { age: 3600 },
  removeOnFail: false,
};
```

Workers run in separate NestJS context (or same process, separate module) — use `@Processor` decorator. In production, workers can be extracted to separate service instances without code changes.

## 7.8 Auto-Completion Background Job

A scheduled job (BullMQ repeatable job, every 1 minute) queries:

```sql
SELECT tp.id FROM tests_participants tp
JOIN tests t ON t.id = tp.test_id
WHERE tp.status = 'IN_PROGRESS'
  AND t.end_time < NOW()
  AND tp.deleted_at IS NULL;
```

For each result: update status to `COMPLETED`, set `completed_at = NOW()`, trigger evaluation for any unscored submissions.

---

# 8. Frontend Implementation Plan

## 8.1 Architecture

Next.js 14+ (App Router) with TypeScript. State management via Zustand (lightweight) for client state; React Query (TanStack Query) for server state / API calls.

```
app/
├── (auth)/
│   ├── login/page.tsx
│   ├── register/page.tsx
│   └── oauth/callback/page.tsx
├── (dashboard)/
│   ├── layout.tsx            # Authenticated layout with nav
│   ├── dashboard/page.tsx
│   ├── tests/
│   │   ├── page.tsx          # Test listing
│   │   ├── create/page.tsx   # Test creation wizard
│   │   └── [id]/
│   │       ├── page.tsx      # Test detail / instructor view
│   │       ├── take/page.tsx # Participant test-taking interface
│   │       └── results/page.tsx
│   ├── questions/
│   │   └── page.tsx          # Question bank
│   ├── analytics/
│   │   └── page.tsx
│   └── admin/
│       ├── users/page.tsx
│       └── roles/page.tsx
└── api/                      # Next.js API routes (BFF layer if needed)
```

## 8.2 Key UI Components

**Test Creation Wizard** (multi-step):
1. Basic info (title, description, timing, capacity)
2. Question config (type, count, generate via LLM or manual)
3. Marking scheme (marks per question, negative marking)
4. Review & publish

LLM generation step: show spinner + polling against `GET /jobs/{jobId}` every 2 seconds. On completion, show generated questions for instructor review/edit before saving.

**Test-Taking Interface**:
- Timer component (countdown to `end_time`, client-side with server sync)
- Question navigator (sidebar showing completion status per question)
- Auto-save answers every 30 seconds to `POST /submissions` (draft mode — `attempt_no` increment means re-submission is allowed)
- Confirm submit modal; disable submit after `end_time`
- Handle network disconnection gracefully — queue submissions locally, retry on reconnect

**Results View**:
- Only accessible after `participant.status = COMPLETED`
- Show per-question score, total score, leaderboard position
- Subjective answers show LLM rationale

## 8.3 Authentication Flow

- Store access token in memory (Zustand store), refresh token in `httpOnly` cookie
- Axios interceptor: on 401, attempt token refresh; on refresh failure, redirect to login
- OAuth callback page: exchange code for tokens via `/auth/oauth/{provider}/callback`, then redirect to dashboard

## 8.4 RBAC on Frontend

- On login, fetch user permissions via `/access-check` or embed in JWT claims (role name only; detailed permissions fetched separately)
- Zustand store holds `permissions: string[]` (resource keys)
- `usePermission('tests:create')` hook gates UI elements
- Server-side route protection via Next.js middleware (check access token validity)

---

# 9. Integration Plan

## 9.1 LLM Integration

**Provider abstraction**:
```typescript
// LLMModule registers provider based on LLM_PROVIDER env var
// Both providers implement LLMProvider interface
// Swap provider without changing calling code
```

**Question generation prompt contract**:
```
System: You are an expert educator. Generate exactly {count} {type} questions about the topic below.
Return a JSON array matching this schema: [{"question_text": string, "type": string, "options": [...] | null, "test_cases": [...] | null, "default_marks": number}]
User: Topic: {description}
Additional requirements: {difficulty}, {language}
```

**Subjective evaluation prompt contract**:
```
System: You are an expert evaluator. Score the following answer.
Return JSON: {"score": number, "max_score": number, "rationale": string, "confidence": float}
User: Question: {question_text}
Answer: [ANSWER START]{participant_answer}[ANSWER END]
Max score: {marks}
```

**Cost controls**:
- Max input tokens per generation: 2000
- Max output tokens per generation: 4000
- Token usage logged per user per month; enforced against subscription plan credits

## 9.2 Judge0 Integration (Coding Evaluation)

```typescript
interface Judge0Submission {
  source_code: string;  // base64 encoded
  language_id: number;  // Judge0 language IDs
  stdin: string;        // base64 encoded
  expected_output: string;
  cpu_time_limit: number;  // seconds
  memory_limit: number;    // KB
}
```

Submit all test cases in parallel batches. Collect results via polling `GET /submissions/batch?tokens=...`. Aggregate: `{passed: N, total: M, results: [...]}` → store in `evaluated_result` jsonb.

Hidden test cases: `is_hidden: true` in `test_cases` jsonb → result shows pass/fail but not input/output.

## 9.3 OAuth Integration

OAuth flow: redirect to provider → callback → exchange code for profile → upsert user in DB → issue JWT.

Provider configuration (via Passport strategies):
- Google: scope `['email', 'profile']`
- GitHub: scope `['user:email']`

Account linking policy (open question — see section 19): defaulting to "auto-link by verified email" in Phase 1.

## 9.4 Email Integration

Use `@nestjs-modules/mailer` with Handlebars templates. Templates:
- `otp-verification.hbs` — OTP code + expiry
- `password-reset.hbs` — reset link
- `test-invitation.hbs` — participant invited to test (future)
- `test-completion.hbs` — results available notification

Queue all email sends through BullMQ `mail` queue (fire-and-forget with retry).

## 9.5 Payment Integration (Stripe)

Integration points:
- `POST /payments/checkout` — create Stripe Checkout Session for plan upgrade
- `POST /payments/webhook` — Stripe webhook handler (verify signature, process events)
- Webhook events to handle: `checkout.session.completed`, `invoice.payment_failed`, `customer.subscription.deleted`
- Idempotency: use Stripe's idempotency keys; store `stripe_subscription_id` on `user_subscriptions` to prevent duplicate processing

---

# 10. Scalability and Performance Considerations

## 10.1 Database

- **Connection pooling**: PgBouncer in transaction mode, pool size = `(CPU_cores × 2) + disk_count`. NestJS TypeORM pool: 10–20 connections per instance.
- **Read replicas**: Analytics queries (leaderboard, usage stats, `/tests/{id}/stats`) routed to read replica via separate `DataSource`.
- **Partitioning** (future, not Phase 1): `submissions` table partitioned by `submitted_at` month when row count exceeds 10M.
- **Vacuum tuning**: `autovacuum_vacuum_scale_factor = 0.05` for `submissions` and `tests_participants` (high-write tables).

## 10.2 Caching Strategy

| Data | Cache key pattern | TTL | Invalidation trigger |
|---|---|---|---|
| RBAC permission map | `rbac:{roleId}` | 60s | PUT /roles/{key}/permissions |
| Leaderboard | `leaderboard:{testId}` | 60s | New submission scored |
| Test detail | `test:{testId}` | 300s | PATCH /tests/{id} |
| User profile | `user:{userId}` | 300s | PATCH /users/{username} |
| Job status | `job:{jobId}` | 86400s | Job completion |

Use Redis `SET ... EX ... NX` for cache population; publish to `cache:invalidate` channel for distributed invalidation across multiple NestJS instances.

## 10.3 Concurrent Test-Taking

**Participant slot reservation** (the critical race condition):

```sql
WITH updated AS (
  UPDATE tests
  SET current_participants = current_participants + 1
  WHERE id = $testId
    AND current_participants < max_participants
  RETURNING id
)
INSERT INTO tests_participants (user_id, test_id, status)
SELECT $userId, $testId, 'IN_PROGRESS'
WHERE EXISTS (SELECT 1 FROM updated)
RETURNING *;
```

If insertion returns 0 rows → test is full → 422 response.

**Submission attempt_no race condition**:

Use PostgreSQL advisory lock per participant during submission:
```sql
SELECT pg_advisory_xact_lock(hashtext('submission:' || $participantId));
-- Then compute MAX(attempt_no) + 1 and insert
```

Or use a `SELECT ... FOR UPDATE` on the `tests_participants` row to serialize per-participant submissions.

## 10.4 LLM Job Throughput

- BullMQ concurrency per worker: 5 (LLM calls are I/O-bound)
- Separate queues for generation (user-facing, high priority) vs. evaluation (background, lower priority)
- BullMQ priority: generation jobs priority=1, evaluation jobs priority=5
- Stagger evaluation: after test completion, dispatch evaluation jobs with 100ms delay between questions to avoid LLM rate limits

## 10.5 Horizontal Scaling

NestJS instances are stateless (JWT, session state in Redis). Scale horizontally behind Nginx load balancer. BullMQ workers can run as separate Deployment in Kubernetes — scale independently based on queue depth.

---

# 11. Security Considerations

## 11.1 Authentication Security

- Passwords: bcrypt with cost factor 12
- JWT: HS256 with 256-bit secrets (separate secrets for access and refresh tokens)
- Refresh token rotation: each refresh issues new token and invalidates old one (family tracking to detect reuse attacks)
- Access token: short-lived (15 min), not stored in cookies (memory-only in frontend)
- Refresh token: stored in `httpOnly, Secure, SameSite=Strict` cookie
- CSRF protection: not required for `httpOnly` cookie with `SameSite=Strict`, but add CSRF token for forms that submit with cookies

## 11.2 Input Security

- All inputs sanitized via `class-validator` whitelist mode
- SQL injection: TypeORM parameterized queries only — never string concatenation in queries
- XSS: all user-generated content escaped on output; Content-Security-Policy header
- File uploads (if added later): validate MIME type, limit size, store in S3 (never serve directly)

## 11.3 Code Execution Security

CODING questions execute untrusted code — critical attack surface:

- Judge0 runs in isolated containers (gVisor or nsjail sandbox)
- CPU time limit: 5 seconds max per test case
- Memory limit: 256MB per execution
- No network access from sandbox
- No filesystem access beyond `/tmp`
- Never run Judge0 on application servers — deploy as separate isolated service

## 11.4 LLM Prompt Injection

- Participant answers for subjective evaluation pass through LLM — prompt injection risk
- Mitigation: wrap participant answer in explicit delimiters; system prompt instructs model to treat content between `[ANSWER START]` and `[ANSWER END]` as untrusted user content
- Log all LLM inputs/outputs for audit trail
- Do not include other participants' data in prompts

## 11.5 API Security

- Helmet.js: sets security headers (X-Frame-Options, X-Content-Type-Options, HSTS)
- CORS: allowlist of known frontend origins
- Rate limiting: per-IP and per-user (see section 6.4)
- Request size limit: 1MB for most endpoints; 10MB for code submission (CODING type)
- API key rotation: document key rotation process for LLM providers; support multiple active keys during rotation

## 11.6 RBAC Security

- Resource keys are server-defined (seeded in DB) — clients cannot create new resource types
- `@RequirePermission` guard runs on every protected route — cannot be bypassed via URL manipulation
- Owner-only checks verified in service layer (not just controller) — defense in depth
- Audit log for all ADMIN operations (role/permission changes, user activation/deactivation)

---

# 12. Reliability and Failure Handling

## 12.1 Failure Scenarios and Responses

| Failure | Detection | Response |
|---|---|---|
| LLM API down during question generation | HTTP 5xx or timeout | Job fails → retry 3x with backoff → dead-letter queue → notify user via email/webhook; test remains in draft |
| LLM API down during evaluation | Same | Submission marked `evaluation_pending`; retry queue; human review fallback |
| Code sandbox (Judge0) down | HTTP timeout | Retry 3x; if still failing, mark submission as `evaluation_failed`; alert ops |
| DB primary failure | Connection error | PgBouncer detects; promote read replica (if using RDS Multi-AZ, automatic); NestJS retries failed queries up to 3x |
| Redis down | Connection error | Auth still works (verify JWT signature); RBAC falls back to DB query; leaderboard cache miss → DB query |
| Participant disconnects mid-test | No heartbeat | Auto-save on frontend every 30s; server-side auto-complete job handles expiry; answers already submitted are preserved |
| Test ends while user is submitting | Race between `end_time` check and submission | Accept submission if received before `end_time + 30s grace period` (configurable) |

## 12.2 Idempotency

- All POST endpoints that create resources: accept optional `X-Idempotency-Key` header; store result in Redis for 24 hours; re-send cached response on duplicate request
- Stripe webhook processing: check `stripe_subscription_id` exists before processing; use Stripe event IDs as idempotency keys

## 12.3 Health Checks

`GET /health` endpoint (not versioned, at `/health`):
```json
{
  "status": "ok",
  "db": "connected",
  "redis": "connected",
  "queues": {"llm-generation": {"waiting": 3, "active": 1}},
  "version": "1.0.0"
}
```

Kubernetes liveness probe: `/health/live` (always 200 if process is running)
Kubernetes readiness probe: `/health/ready` (200 only if DB and Redis connected)

## 12.4 Circuit Breakers

Implement circuit breakers for external dependencies (LLM API, Judge0, SMTP) using `opossum` library:
- Threshold: 50% failure rate over 10-second window → OPEN circuit
- Half-open after 30 seconds (single test request)
- Metrics exposed to monitoring

---

# 13. Observability and Monitoring

## 13.1 Structured Logging

Use `pino` (via `nestjs-pino`) for structured JSON logs. Every log entry includes:
```json
{
  "timestamp": "ISO8601",
  "level": "info|warn|error",
  "requestId": "uuid",
  "userId": "uuid|null",
  "module": "TestsModule",
  "method": "createTest",
  "durationMs": 142,
  "message": "..."
}
```

Never log: passwords, tokens, PII beyond userId, LLM API keys.

## 13.2 Distributed Tracing

OpenTelemetry SDK initialized at `main.ts` bootstrap. Instrument:
- All HTTP requests (NestJS OpenTelemetry plugin)
- All TypeORM queries (TypeORM OpenTelemetry plugin)
- All BullMQ job executions
- All LLM API calls (custom span: `llm.generate`, `llm.evaluate`)
- All Redis operations

Export to Datadog/Jaeger. Trace ID propagated in `X-Trace-ID` response header.

## 13.3 Metrics

Expose Prometheus metrics at `/metrics` (protected by internal network ACL):

| Metric | Type | Labels |
|---|---|---|
| `http_requests_total` | Counter | method, path, status_code |
| `http_request_duration_seconds` | Histogram | method, path |
| `db_query_duration_seconds` | Histogram | operation, table |
| `llm_requests_total` | Counter | provider, operation, status |
| `llm_tokens_total` | Counter | provider, type (prompt/completion) |
| `queue_job_duration_seconds` | Histogram | queue, status |
| `queue_depth` | Gauge | queue |
| `active_participants` | Gauge | — |

## 13.4 Alerts

| Alert | Condition | Severity |
|---|---|---|
| High error rate | 5xx rate > 1% over 5 min | P1 |
| LLM queue depth | queue depth > 100 for > 5 min | P2 |
| Evaluation failures | dead-letter queue non-empty | P2 |
| DB connection exhaustion | available connections < 5 | P1 |
| Slow queries | p99 query latency > 2s | P2 |
| Redis memory usage | > 80% | P2 |

## 13.5 Audit Log

Write to dedicated `audit_logs` table (append-only):
```sql
CREATE TABLE audit_logs (
  id BIGSERIAL PRIMARY KEY,
  actor_id UUID,
  action VARCHAR NOT NULL,
  resource_type VARCHAR,
  resource_id VARCHAR,
  old_value JSONB,
  new_value JSONB,
  ip_address INET,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

Write to this table for: all ADMIN actions, role/permission changes, user deactivation, test deletion.

---

# 14. Testing Strategy

## 14.1 Test Pyramid

```
         [E2E Tests]          ~5%   Playwright: critical user journeys
        [Integration]        ~25%   Supertest: controller → DB (real PG via testcontainers)
       [Unit Tests]          ~70%   Jest: services, validators, utils, LLM parsers
```

## 14.2 Unit Tests

Each service has a corresponding `.spec.ts`. Mock all external dependencies (repositories, LLM provider, Redis).

Priority test cases:
- `ParticipantService.enroll()`: test full, double enroll, valid enroll
- `SubmissionService.submit()`: after end_time, not IN_PROGRESS, valid submit, attempt_no increment
- `EvaluationLogic.scoreMCQ()`: correct answer, incorrect answer, negative marking
- `LLMModule.parseGeneratedQuestions()`: valid JSON, malformed JSON, missing fields
- `RBACGuard.canActivate()`: permission granted, permission denied, unauthenticated

## 14.3 Integration Tests

Use `@nestjs/testing` + `testcontainers` to spin up real PostgreSQL and Redis instances. Test:
- Full auth flow: register → login → refresh → logout
- Concurrent enrollment: 10 parallel requests to enroll in a test with `max_participants = 5` — verify exactly 5 succeed
- Submission attempt_no: concurrent submissions — verify `attempt_no` is sequential with no gaps/duplicates
- Soft delete: deleted user not returned in list queries

## 14.4 E2E Tests

Playwright tests for:
- Test creation wizard (manual + LLM generation with mocked LLM)
- Full test-taking journey: enroll → start → submit all questions → complete → view results
- Admin user management flow
- RBAC: verify instructor cannot access admin endpoints

## 14.5 LLM Testing

LLM calls are mocked in unit and integration tests. For LLM-specific tests:
- Prompt snapshot tests: verify prompt templates render correctly for given inputs
- Output parser tests: test against a library of real LLM responses (good and malformed)
- Contract tests: verify LLM responses conform to expected JSON schema (run against real API in staging only)

## 14.6 Load Testing

Use k6 for load testing before production launch:
- Scenario 1: 100 concurrent participants taking same test — submit answers continuously for 30 min
- Scenario 2: 50 concurrent test creation requests (with LLM mocked)
- Scenario 3: Leaderboard endpoint under 500 concurrent reads
- Acceptance criteria: p99 latency < 500ms for all endpoints; zero 5xx errors under normal load

---

# 15. Rollout and Deployment Strategy

## 15.1 Environment Setup

| Environment | Purpose | Infrastructure |
|---|---|---|
| `development` | Local dev | docker-compose (PG + Redis + BullMQ UI) |
| `staging` | Integration testing, QA | Single ECS task per service, RDS t3.medium, ElastiCache |
| `production` | Live system | ECS with auto-scaling, RDS Multi-AZ r6g.large, ElastiCache cluster |

## 15.2 CI/CD Pipeline

```
PR opened
  → Lint (ESLint + Prettier)
  → Type check (tsc --noEmit)
  → Unit tests (Jest)
  → Build (docker build)

Merge to main
  → All above +
  → Integration tests (testcontainers)
  → Push to ECR
  → Deploy to staging (ECS rolling update)
  → E2E smoke tests against staging

Tag release (vX.Y.Z)
  → Deploy to production (Blue/Green via CodeDeploy)
  → Run DB migrations (pre-deploy step)
  → Smoke tests
  → Monitor error rates for 15 min → auto-rollback if > threshold
```

## 15.3 Database Migration Deployment

Migrations run before new application code is deployed (pre-deploy hook). All migrations must be backward-compatible with the previous application version (no removing columns that old code reads, no adding `NOT NULL` without defaults).

Migration checklist:
1. New column: add as nullable, backfill, add NOT NULL constraint in separate migration
2. Rename column: add new column, dual-write in app, backfill, remove old column in next release
3. Index creation: use `CREATE INDEX CONCURRENTLY` (non-blocking)
4. Never drop columns in same release that removes their usage — always one release lag

## 15.4 Feature Flags

Use environment-variable-based feature flags for risky features:
```
FEATURE_LLM_GENERATION=true
FEATURE_CODING_QUESTIONS=false
FEATURE_PAYMENTS=false
```

`ConfigService.isFeatureEnabled('llm_generation')` gates both API routes and UI. Allows gradual rollout and instant kill-switch without redeploy.

---

# 16. Phased Execution Plan

## Phase 0: Foundation (Sprint 1–2, ~2 weeks)

**Objective**: Runnable skeleton with auth and data layer. No business logic yet.

**Backend tasks**:
- NestJS project scaffold with module structure, global pipes, filters, interceptors
- TypeORM setup with PostgreSQL connection, base entities
- All migrations: users, roles, users_roles, resources, permissions (schema only)
- Seed migrations: default roles (ADMIN, INSTRUCTOR, PARTICIPANT), default resources, default permissions
- Redis connection module (ioredis)
- BullMQ setup with UI dashboard (bull-board)
- ConfigModule with Zod env validation
- Health check endpoints
- Docker-compose for local dev (PG, Redis, BullMQ UI)

**Auth tasks**:
- `POST /auth/register` (local, bcrypt password)
- `POST /auth/login` (issue access + refresh tokens)
- `POST /auth/refresh` (rotate refresh token)
- `POST /auth/logout` (blacklist refresh token in Redis)
- JwtAuthGuard + @CurrentUser decorator

**RBAC tasks**:
- RBACGuard implementation
- `GET /roles`, `POST /roles`
- `GET /permissions`
- `PUT /roles/{roleKey}/permissions`

**Frontend tasks**:
- Next.js project scaffold (App Router, TypeScript, Tailwind)
- Auth layout + login/register pages
- Axios instance with interceptors (token refresh)
- Zustand store: auth state, permissions
- Basic dashboard layout (nav, protected routes)

**DB tasks**:
- Write all Phase 0 migrations
- PgBouncer configuration for staging

**API tasks**:
- Implement ResponseEnvelopeInterceptor
- Implement GlobalExceptionFilter
- Set up API versioning (`/api/v1` prefix)
- Helmet, CORS, throttler global setup

**Dependencies**: None (foundational)

**Risks**: JWT secret rotation strategy not defined — define in this sprint.

**Testing**:
- Unit tests for AuthService (login, register, refresh, logout)
- Integration test: full auth flow via Supertest
- Manual smoke test: register → login → access protected route

**Rollout considerations**: Deploy to staging only. No external traffic.

---

## Phase 1: Core Test & Question Management (Sprint 3–4, ~2 weeks)

**Objective**: Instructors can create, configure, and publish tests with questions (manual only — no LLM yet).

**Backend tasks**:
- Migrations: tests, questions, tests_questions
- `TestRepository`: CRUD with filter support (status, created_by, search, sort, popular)
- `TestService`: create, update, delete (validate edit before start_time)
- `QuestionRepository` + `QuestionService`: CRUD with filters
- `POST /tests/{id}/questions`, `PATCH /tests/{id}/questions/{qid}`, `DELETE /tests/{id}/questions/{qid}`
- Question type validation (MCQ requires options, CODING requires test_cases)
- All indexes from section 5.2 for these tables

**Frontend tasks**:
- Test listing page with filters (status, search, sort)
- Test creation form (basic info + manual question addition)
- Question bank page
- Test detail page (instructor view: questions, marking scheme)

**DB tasks**:
- Migrations for tests, questions, tests_questions
- Indexes: all test and question indexes from section 5.2

**API tasks**:
- All `/tests/*` and `/questions/*` endpoints
- Implement pagination (offset-based for now)
- Implement `unused` filter for questions

**Dependencies**: Phase 0 complete (auth, RBAC)

**Risks**: Test status derivation logic (NOW() comparisons) is timezone-sensitive — store all timestamps in UTC, validate in tests.

**Testing**:
- Unit tests: TestService (create, update, delete, status derivation)
- Unit tests: QuestionService (type validation, CRUD)
- Integration tests: test creation → add questions → fetch test with questions
- Authorization tests: PARTICIPANT cannot create test, INSTRUCTOR cannot edit others' tests

**Rollout considerations**: Internal release to beta instructors for feedback on test creation UX.

---

## Phase 2: Participant Enrollment and Test-Taking (Sprint 5–6, ~2 weeks)

**Objective**: Participants can enroll in tests, take them, and submit answers. Auto-evaluation for MCQ.

**Backend tasks**:
- Migration: tests_participants, submissions
- `ParticipantService`: enroll (atomic slot reservation), state transitions
- `SubmissionService`: submit answer, compute attempt_no (with advisory lock)
- `EvaluationModule`: MCQ auto-scoring synchronous path
- Submission visibility gate (only visible after COMPLETED)
- Auto-complete background job (BullMQ repeatable)
- `GET /tests/{id}/participants`, `GET /users/{id}/tests`

**Frontend tasks**:
- Test-taking interface (timer, question navigator, answer inputs for MCQ/subjective)
- Auto-save answers (debounced, 30s interval)
- Submit confirmation modal with warning
- Results page (per-question scores, total, pending for subjective)

**DB tasks**:
- Migrations: tests_participants, submissions + all indexes
- Advisory lock setup documentation
- Unique index: `(user_id, test_id)` with `WHERE deleted_at IS NULL AND status != 'COMPLETED'`

**API tasks**:
- `POST /tests/{id}/participants` (enroll with atomic slot reservation)
- `POST /submissions` (submit answer, trigger evaluation)
- `GET /submissions/{id}`, `GET /participants/{id}/submissions`
- `GET /tests/{id}/submissions` (instructor view)

**Dependencies**: Phase 1 complete

**Risks**: Concurrent enrollment race condition — load test with 10+ parallel enroll requests before release.

**Testing**:
- Concurrency test: 10 parallel enroll requests, max_participants=5 — verify exactly 5 enrolled
- Concurrency test: parallel submissions — verify attempt_no sequential
- Integration: full happy-path flow (enroll → start → submit MCQ → complete → view results)
- Edge cases: submit after end_time, double enroll, submit without IN_PROGRESS status

**Rollout considerations**: Load test concurrent enrollment before exposing to real users.

---

## Phase 3: LLM Integration — Question Generation (Sprint 7–8, ~2 weeks)

**Objective**: Instructors can generate questions via LLM from a topic/description. Subjective evaluation via LLM.

**Backend tasks**:
- `LLMModule` with provider abstraction (OpenAI first, Anthropic as secondary)
- Prompt templates (Handlebars) for question generation and subjective evaluation
- BullMQ workers: `llm-generation`, `llm-evaluation`
- Job status polling endpoint: `GET /jobs/{jobId}`
- SSE endpoint: `GET /jobs/{jobId}/stream`
- `POST /questions` with `{generate: true}` flag dispatches to queue
- LLM output parser with Zod schema validation and retry on malformed output
- Subjective evaluation worker: triggered after participant completes test
- Rate limiting for LLM endpoints (10 req/hour per user)
- Token usage logging

**Frontend tasks**:
- LLM generation step in test creation wizard
- Polling/SSE for job status with progress indicator
- Generated question review/edit UI (allow instructor to modify before saving)
- Subjective answer results: show LLM rationale + score (after completion)

**DB tasks**:
- No schema changes; `evaluated_result` jsonb already present in submissions

**API tasks**:
- `POST /questions?generate=true` (returns jobId)
- `GET /jobs/{jobId}` (poll)
- `GET /jobs/{jobId}/stream` (SSE)

**Dependencies**: Phase 2 complete, BullMQ setup from Phase 0, LLM API keys in env

**Risks**:
- LLM output non-determinism — parser must handle format variations
- Cost overrun — implement hard limit per user per day; alert at 80% of daily budget

**Testing**:
- Unit tests: prompt template rendering, output parser (valid + malformed JSON)
- Integration tests: LLM mocked — verify job dispatch, completion, question persistence
- Contract tests (staging only): real LLM call → validate response schema

**Rollout considerations**: Enable via feature flag `FEATURE_LLM_GENERATION=true`. Monitor token costs daily.

---

## Phase 4: Coding Questions and Analytics (Sprint 9–10, ~2 weeks)

**Objective**: CODING question type fully functional with sandbox execution. Analytics and leaderboard live.

**Backend tasks**:
- Judge0 integration module (HTTP client with retry + circuit breaker)
- BullMQ worker: `code-execution`
- Coding evaluation: batch submit test cases, collect results, compute score
- Hidden test case handling (result shows pass/fail, not input/output)
- `AnalyticsModule`: test stats, user stats, leaderboard
- Leaderboard query with Redis cache (60s TTL)
- `GET /analytics/leaderboard`, `GET /analytics/usage`
- `GET /tests/{id}/stats`, `GET /users/{id}/stats`

**Frontend tasks**:
- Code editor component (Monaco Editor) for CODING questions
- Language selector, test case input UI
- Leaderboard page with pagination
- Analytics dashboard (for ADMIN/INSTRUCTOR)

**DB tasks**:
- Materialized view for test_stats (precomputed aggregations)
- Read replica routing for analytics queries

**API tasks**:
- All analytics endpoints
- Coding submission: language param in submission payload

**Dependencies**: Phase 3 complete, Judge0 instance deployed

**Risks**:
- Judge0 self-hosting complexity — consider Judge0 cloud as fallback
- Sandbox security — verify isolation before production

**Testing**:
- Integration: submit code → queue → Judge0 (mocked) → score persisted
- Concurrency: multiple coding submissions for same participant
- Analytics: verify leaderboard ordering (highest score, earliest completion as tiebreaker)
- Performance: leaderboard query < 200ms with cache; < 2s without cache

**Rollout considerations**: Enable CODING questions via feature flag. Security audit of Judge0 sandbox before prod.

---

## Phase 5: OAuth, OTP, and Advanced Auth (Sprint 11, ~1 week)

**Objective**: Complete auth system — OAuth, OTP, password reset, brute force protection.

**Backend tasks**:
- Google OAuth strategy (Passport)
- GitHub OAuth strategy (Passport)
- OTP generation, hashing, Redis storage, email send
- `POST /auth/otp`, `POST /auth/otp/verify`
- Password forgot/reset flow (Redis token storage)
- Brute force lockout (Redis counter per IP/email)
- Refresh token family tracking

**Frontend tasks**:
- OAuth buttons (Google, GitHub) on login/register pages
- OTP input UI
- Forgot password / reset password pages

**DB tasks**:
- No schema changes (OAuth fields already in users table)

**Dependencies**: Phase 0 complete, SMTP provider configured, Google/GitHub OAuth apps registered

**Testing**:
- Unit: OTP generation/verification, token expiry
- Integration: OAuth callback flow with mocked provider
- Security: brute force test — verify lockout after 5 attempts

**Rollout considerations**: Register OAuth apps in staging environment first. Confirm redirect URIs before production.

---

## Phase 6: User Management and Payments (Sprint 12–13, ~2 weeks)

**Objective**: Complete user management CRUD. Payment/subscription integration.

**Backend tasks**:
- Full UsersModule: CRUD, soft delete, activate/deactivate/restore, filters
- `PaymentsModule`: Stripe Checkout, webhook handler, subscription lifecycle
- Subscription-gated feature enforcement (e.g., LLM credit check before generation)
- `POST /payments/checkout`, `POST /payments/webhook`

**Frontend tasks**:
- User management page (ADMIN)
- Pricing/upgrade page
- Stripe Checkout redirect integration
- Subscription status in user profile

**DB tasks**:
- Migrations: subscription_plans, user_subscriptions (schema from section 5.3)

**Dependencies**: Phase 0 (auth), Stripe account + API keys, payment schema approved by team

**Testing**:
- Webhook handling: test all Stripe event types with Stripe CLI
- Subscription enforcement: verify LLM generation blocked when credits exhausted
- Edge case: simultaneous subscription upgrades from same user

**Rollout considerations**: Test with Stripe test mode keys for 2 weeks before switching to live keys.

---

## Phase 7: Hardening and Production Launch (Sprint 14–15, ~2 weeks)

**Objective**: Production-ready — performance, security, observability, load testing.

**Tasks**:
- OpenTelemetry instrumentation (all modules)
- Prometheus metrics endpoint + Grafana dashboards
- Alert rules (from section 13.4)
- k6 load tests (scenarios from section 14.6)
- Security review: RBAC audit, prompt injection review, input sanitization audit
- PgBouncer tuning, connection pool sizing
- Redis Sentinel/Cluster setup for production
- Blue/Green deployment pipeline finalization
- Runbook documentation for common failure scenarios
- Penetration test (external, lightweight scope: OWASP Top 10)
- `audit_logs` table and ADMIN action logging wired up
- Rate limiting tuning based on load test results

**Dependencies**: All phases complete

**Testing**:
- Full regression E2E suite
- Load test: all three scenarios from section 14.6
- Chaos test: kill Redis → verify auth still works; kill DB replica → verify analytics falls back gracefully

**Rollout considerations**: Soft launch to internal team + invited beta users. Monitor for 1 week before public launch.

---

# 17. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| LLM output quality insufficient for question generation | Medium | High | Build review step — instructors always review before saving; provide regenerate option |
| LLM API cost exceeds budget | Medium | Medium | Per-user credit limits; hard stop at daily budget; cost dashboard |
| Judge0 availability in production | Medium | High | Self-host on dedicated VM; fallback to Judge0 cloud; circuit breaker |
| PostgreSQL write bottleneck under concurrent submissions | Low | High | Connection pooling, advisory locks, async evaluation; load test in Phase 2 |
| Refresh token reuse attack (token theft) | Low | High | Token family tracking; detect and revoke all tokens in family on reuse |
| Subjective evaluation inconsistency (LLM scores vary) | High | Medium | Store rationale; add human review flag for low-confidence scores; allow instructor override |
| OAuth account linking — duplicate accounts | Medium | Medium | Enforce email uniqueness at DB level; auto-link by verified email policy; account merge UI |
| Migration failure during deployment | Low | Critical | Always backward-compatible migrations; blue/green deployment; migration rollback script per migration |
| Feature creep in LLM prompts causing test failures | Medium | Medium | Prompt versioning; snapshot tests; staging-only real-LLM tests |
| Payment webhook missed or processed twice | Low | High | Stripe idempotency keys; process-once check via `stripe_subscription_id` uniqueness |
| Coding sandbox security escape | Low | Critical | gVisor/nsjail isolation; network isolation; run in separate VPC subnet; regular sandbox updates |
| `current_participants` drift from actual row count | Low | Medium | Periodic reconciliation job; DB constraint as final guard |

---

# 18. Assumptions

1. **TypeORM** is the chosen ORM (not Prisma). If switching to Prisma, migration files differ and the `DataSource.transaction()` API changes.
2. **PostgreSQL 15+** is used — required for `gen_random_uuid()` default and JSON path operators.
3. **JWT-based auth** with no server-side session storage (except refresh tokens in Redis). Stateless access tokens.
4. **Single region deployment** for Phase 1. Multi-region read replicas are a future concern.
5. **OAuth account linking by email**: if a user registers via Google and later tries GitHub with the same email, accounts are auto-linked. This policy must be confirmed.
6. **Instructor owns questions**: questions created by an instructor are personal to them (not shared) unless explicitly attached to a test visible to others.
7. **Test cannot be edited after start_time**: any edit (questions, marking, timing) is blocked once the test has started. This prevents mid-test inconsistencies.
8. **All timestamps in UTC** at the database level. Frontend displays in user's local timezone via JavaScript.
9. **Enrollment is implicit with test-start**: no separate "pre-registration" step. `tests_participants` row is created when user starts the test (not before). Confirm if a pre-enrollment/invitation flow is needed.
10. **LLM provider is interchangeable**: the same prompt contract works for OpenAI and Anthropic. Prompt tuning may be needed per provider.
11. **Judge0 handles multiple languages**: the specific language list (Python, Java, JS, C++, etc.) is defined in `test_cases` jsonb with a `language_id` field matching Judge0's language registry.
12. **Payment gating on LLM credits**: free tier gets N questions generated per month; paid plans get more. The exact credit limits are TBD with product.
13. **Frontend is a separate repository/deployment** from the NestJS backend. BFF (Backend for Frontend) pattern via Next.js API routes is optional.
14. **`tests` table has no soft delete**: confirmed by absence of `deleted_at` column in ERD v2.2.1. Hard delete cascades to `tests_questions` and `tests_participants`. Validate this is intentional.

---

# 19. Open Questions

1. **Pre-enrollment vs. on-demand start**: Does the platform support a model where participants pre-register for a test (and the instructor sees the list before the test starts), or do they simply show up during the test window? The current schema supports in-progress-on-start, but the `tests_participants` row could also be created on explicit enrollment with `NOT_STARTED` status.

2. **OAuth account linking policy**: If user A registers with Google (email: a@b.com) and later tries GitHub with the same email — auto-link, reject, or show a merge UI? Define before AuthModule Phase 5 implementation.

3. **Test editing after publish**: Can an instructor edit marking scheme or add questions after the test has started? Current assumption is no, but some platforms allow editing future/unstarted questions only.

4. **Subjective evaluation human override**: Can an instructor manually override the LLM-assigned score for a subjective answer? The schema supports it (update `submissions.score`), but the UI and permission model needs to be defined.

5. **Multiple test attempts per user**: Can a user retake the same test? Current schema blocks two `IN_PROGRESS` rows, but after completion, can they start a new `tests_participants` row? If yes, leaderboard logic (first-attempt only) needs to change.

6. **Notification system**: Is there an in-app notification system or just email? Test completion, invitation, results-available events need delivery channels defined.

7. **Test invitation flow**: Can instructors invite specific users to a private test (not open enrollment)? The current schema has no concept of invite tokens or test visibility (public/private). This affects the participants module design significantly.

8. **Maximum test duration vs. window**: Tests have `start_time`, `end_time`, and `duration`. If `duration` is 60 min but `end_time - start_time` is 2 hours, does the test end at `started_at + duration` or `end_time`? Define precedence — this directly affects auto-complete logic.

9. **Coding question language constraints**: Can a test restrict which programming languages are allowed for CODING questions? Currently not in schema.

10. **Subscription plan limits enforcement**: Which operations are gated behind paid plans? (Assumed: LLM generation is gated.) Are there limits on test count, question count, or participant count per plan? Required before PaymentsModule design is finalized.

11. **Analytics retention**: How long is submission data retained? Are there GDPR/data-retention requirements for user data and answers?

12. **LLM model pinning**: Should the specific LLM model version (e.g., `gpt-4o-2024-08-06`) be pinned per prompt, or always use the latest? Pinning prevents unexpected behavior changes when providers update models, but requires manual update for improvements.

13. **`tests` hard delete cascade**: Deleting a test hard-deletes all associated `tests_questions`, `tests_participants`, and `submissions`. Is this acceptable from a data-retention and audit perspective? Consider soft-delete for tests if historical data must be preserved.
