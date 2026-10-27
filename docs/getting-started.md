---
title: Getting Started
description: Set up and run the TestFlow backend locally - tools, environment variables, migrations, seeds, tests
status: stable
last_updated: 2026-10-27
---

# Getting Started

This page takes you from a fresh clone to a running backend with data in it. It also lists every environment variable the code reads.

---

## 1. Prerequisites

| Tool | Version | Why you need it | Required? |
| --- | --- | --- | --- |
| Node.js + npm | 20 (the Docker image uses `node:20-alpine`) | Runs the NestJS app, scripts, and tests. | Yes |
| PostgreSQL | 15 (compose uses `postgres:15-alpine`) | Main database. | Yes |
| Redis | 7 | Rate-limit buckets and the read cache. The app still starts if Redis is down (both uses "fail open"). | Strongly recommended |
| RabbitMQ | 4 with the management plugin | Background jobs: AI generation, evaluation, roadmaps, CSV invite import. | Yes (the app connects at boot) |
| Docker + Docker Compose | Any recent | Easiest way to run Postgres, Redis, and RabbitMQ. | Optional |
| An LLM API key | OpenAI by default | AI generation, descriptive grading, roadmaps, assistant. | Needed for AI features |
| Code runner | Piston (self-hosted, separate `piston` repo) or an OnlineCompiler.io key | Running and judging coding questions. | Needed for coding questions |
| SMTP server | Any (Mailtrap, Gmail app password, etc.) | Verification, reset, and invite emails. | Its env vars are required at boot |
| S3 bucket | AWS S3, or MinIO / LocalStack locally | Stores uploaded invitation CSVs. | Its env vars are required at boot |
| Razorpay test keys | - | Checkout and payment webhooks. | Key ID and secret are required at boot |

!!! note "Required at boot"
    Some services read their variables with `getOrThrow` in their constructor, so the app will not start without them, even if you never use that feature. These are: the `DATABASE_*` vars, `RABBITMQ_URL`, `SMTP_HOST`, `SMTP_USER`, `SMTP_PASSWORD`, `EMAIL_FROM`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_BUCKET`, `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, and `GOOGLE_OAUTH2_CLIENT_ID`. The JWT variables are needed as soon as anyone logs in. Put a placeholder value in when you do not need a feature.

---

## 2. Quick start

```bash
git clone <backend repo url> Test-Flow-Backend
cd Test-Flow-Backend
npm ci
cp .env.example .env          # then fill in the values (see section 3)
```

### Option A: run only the services in Docker, the app on your machine

```bash
docker compose up -d db redis rabbitmq
```

The compose file maps the ports like this, so update `.env` to match:

| Service | Host port | Notes |
| --- | --- | --- |
| Postgres | `5434` | Set `DATABASE_PORT=5434`, `DATABASE_HOST=localhost`. |
| Redis | `6380` | Set `REDIS_PORT=6380`. |
| RabbitMQ | `5672` (AMQP), `15672` (UI) | `RABBITMQ_URL=amqp://guest:guest@localhost:5672`. Management UI at `http://localhost:15672` (guest/guest). |

Then:

```bash
npm run migration:run
npm run seed
npm run dev          # nest start --watch, on PORT (default 3000)
```

### Option B: run everything in Docker

```bash
docker compose up --build
```

This builds the `dev` stage of the `Dockerfile` and mounts `./src` and `./tests`, so code changes hot-reload. `entrypoint.dev.sh` waits for Postgres, runs `npm run migration:run`, then `npm run dev`. The compose file overrides `DATABASE_HOST`, `REDIS_HOST`, and `RABBITMQ_URL` so the app can reach the other containers. If you add an npm dependency, run `docker compose up --build` again.

| File | Used for |
| --- | --- |
| `docker-compose.yml` | Local full stack (app + db + redis + rabbitmq). Also used by the CI smoke test. |
| `docker-compose.dev.yml` | The shared dev server. It pulls a prebuilt image and has **no** database service (the dev DB is external). |
| `Dockerfile` | Multi-stage. `dev` stage for hot reload; the last (runtime) stage ships only `dist/` and production deps. |
| `entrypoint.sh` | Runtime image: wait for Postgres, `migration:run:prod`, `start:prod`. |

### Check that it works

| URL | What you should see |
| --- | --- |
| `GET http://localhost:3000/ping` | A public ping response. |
| `GET http://localhost:3000/health` | Status of `database`, `redis`, and `rabbitmq`. |
| `http://localhost:3000/api/docs` | Swagger UI (asks for basic auth, see below). |

---

## 3. Environment variables

The app loads `.env` through `ConfigModule.forRoot({ isGlobal: true })` in `src/app.module.ts`. Code reads values through `ConfigService`. A few places read `process.env` directly because they run before Nest starts: `src/main.ts` (port, Swagger), `src/config/database/data-source.ts` (the TypeORM CLI), and the scripts.

!!! warning "Never commit real secrets"
    `.env` is git-ignored. Only `.env.example` is committed, and it must hold placeholders only. Ask a teammate for dev keys.

### App and docs

| Variable | Meaning |
| --- | --- |
| `PORT` | HTTP port. Default `3000`. |
| `NODE_ENV` | Environment name. The question generator checks for the exact value `testing`: then it picks random questions from the bank instead of calling an LLM (see the note below). |
| `SWAGGER_USER` / `SWAGGER_PASS` | Basic-auth login for `/api/docs`. |
| `FRONTEND_URL` | Frontend base URL, used for links in emails. Default `https://app.testflow.com`. |
| `LOG_LEVEL` | Listed in `.env.example` but not read by any code today. |

!!! note "`NODE_ENV=test` vs `testing`"
    `.env.example` says `test` turns off LLM calls, but `src/assessments/questions/questions.service.ts` checks for `testing`. Use `testing` if you want to skip the LLM.

### Database (PostgreSQL)

| Variable | Meaning |
| --- | --- |
| `DATABASE_HOST` | Host name. |
| `DATABASE_PORT` | Port (`5434` if you use the compose Postgres from your machine). |
| `DATABASE_USERNAME` / `DATABASE_PASSWORD` | Login. |
| `DATABASE_NAME` | Database name (compose default `testflow`). |
| `DATABASE_SSL` | `true` to connect over SSL (with `rejectUnauthorized: false`). |

### Auth (JWT and Google)

| Variable | Meaning |
| --- | --- |
| `JWT_ACCESS_SECRET` | Signs access tokens. Also used to check WebSocket tokens. |
| `JWT_REFRESH_SECRET` | Signs refresh tokens. |
| `JWT_ACCESS_EXPIRY` | Access token lifetime **in seconds** (the code converts the value with `+value`). |
| `JWT_REFRESH_EXPIRY` | Refresh token lifetime **in seconds**. |
| `JWT_EMAIL_VERIFICATION_SECRET` / `JWT_EMAIL_VERIFICATION_EXPIRY` | Secret and lifetime (seconds) of the email verification link. |
| `JWT_PASSWORD_RESET_SECRET` / `JWT_PASSWORD_RESET_EXPIRY` | Secret and lifetime (seconds) of the password reset link. Keep it short. |
| `GOOGLE_OAUTH2_CLIENT_ID` | Google client ID, used as the audience when verifying Google ID tokens on `POST /v1/auth/google`. |

!!! warning "Use plain numbers for JWT expiry"
    `.env.example` shows `15m` and `7d` for the access and refresh expiry, but `src/identity/auth/auth.service.ts` turns them into numbers. `+"15m"` is `NaN`, so use seconds, for example `900` and `604800`.

### Redis

| Variable | Meaning |
| --- | --- |
| `REDIS_HOST` / `REDIS_PORT` | Shared by both Redis connections. |
| `REDIS_PASSWORD` | Leave blank for a local Redis with no auth. |
| `REDIS_DB` | Logical DB for rate-limit buckets. Default `0`. |
| `REDIS_COMMAND_TIMEOUT_MS` | Per-command timeout for the rate limiter. Default `200`. |
| `REDIS_CACHE_DB` | Logical DB for the read cache. Default `1`. Separate so a cache flush never wipes rate limits. |
| `REDIS_CACHE_COMMAND_TIMEOUT_MS` | Per-command timeout for the cache. Default `500`. |

### RabbitMQ

| Variable | Meaning |
| --- | --- |
| `RABBITMQ_URL` | AMQP URL, for example `amqp://guest:guest@localhost:5672`. |
| `CONSUMER` | `true` (default) means this instance also consumes queues. `false` means it only publishes. Use `false` for an API-only instance. |

### LLM providers

Models live in the `ai_models` table. Each row stores the **name** of the env var that holds its key (`apiKeyVariable`) and, optionally, its base URL (`baseUrlVariable`). `src/ai/ai-model-registry.service.ts` reads those names at runtime. So the variables below only matter if a row points at them.

| Variable | Meaning |
| --- | --- |
| `OPENAI_API_KEY` | OpenAI key (used by the default seeded model). |
| `OPENAI_MODEL` | Model code the seeder registers as the default `ai_models` row. Default `gpt-4`. |
| `OPENAI_BASE_URL` | Optional base URL for an OpenAI-compatible host. |
| `ANTHROPIC_API_KEY` | Anthropic key, if you add an Anthropic model row. |
| `GOOGLE_API_KEY` | Google GenAI key, if you add a Google model row. |

### Code execution

| Variable | Meaning |
| --- | --- |
| `CODE_RUNNER_PROVIDER` | `online-compiler` (default) or `piston`. |
| `ONLINE_COMPILER_API_KEY` | OnlineCompiler.io REST key. If unset, runs are refused. |
| `ONLINE_COMPILER_BASE_URL` | Optional override. |
| `ONLINE_COMPILER_MAX_CONCURRENCY` | Parallel runs per key (the API allows 4). |
| `ONLINE_COMPILER_TIMEOUT_MS` | HTTP timeout. Default `40000`. |
| `PISTON_API_KEY` | Raw secret shared with your Piston server (the runner sends its SHA-256). If unset, runs are refused. |
| `PISTON_BASE_URL` | Default `http://127.0.0.1:2000`. From Docker on a Mac, use `http://host.docker.internal:2000`. |
| `PISTON_MAX_CONCURRENCY` | Parallel runs. Default `4`. |
| `PISTON_TIMEOUT_MS` | HTTP timeout. Default `60000`. |
| `PISTON_MAX_RUN_TIMEOUT_MS` / `PISTON_MAX_RUN_CPU_TIME_MS` / `PISTON_MAX_RUN_MEMORY_MB` | Piston's own per-run limits. Requests are clamped to these so Piston does not reply 400. |

### Email (SMTP)

| Variable | Meaning |
| --- | --- |
| `SMTP_HOST` | SMTP server. |
| `SMTP_PORT` | Default `587`. |
| `SMTP_SECURE` | `true` for TLS on connect (usually port 465). |
| `SMTP_USER` / `SMTP_PASSWORD` | SMTP login. |
| `EMAIL_FROM` | Sender address. |

### Payments (Razorpay)

| Variable | Meaning |
| --- | --- |
| `RAZORPAY_KEY_ID` / `RAZORPAY_KEY_SECRET` | API keys used to create orders. |
| `RAZORPAY_WEBHOOK_SECRET` | Secret used to verify the `x-razorpay-signature` header on `POST /webhooks/razorpay`. |
| `CHECKOUT_ORDER_TTL_MINUTES` | Minutes a pending checkout order stays valid. |

### Newsletters

| Variable | Meaning |
| --- | --- |
| `NEWSLETTER_SCHEDULER_ENABLED` | `false` stops the every-minute cron from sending. Use it on any machine that shares a DB with real subscribers. |
| `NEWSLETTER_DISPATCH_BATCH_SIZE` | Due newsletters claimed per pass. Default `5`. |
| `NEWSLETTER_RECIPIENT_BATCH_SIZE` | Recipients per BCC email. Default `100`. |

### File storage (S3)

| Variable | Meaning |
| --- | --- |
| `AWS_REGION` | Bucket region. |
| `AWS_S3_BUCKET` | Bucket name. |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Credentials. |
| `AWS_S3_ENDPOINT` | Blank for AWS. Set it (for example `http://localhost:9000`) for MinIO or LocalStack; path-style URLs are turned on automatically. |

### Scripts only

| Variable | Meaning |
| --- | --- |
| `DEFAULT_PASSWORD_EXPORT_PATH` | Where the `add-default-password-to-users` migration writes its CSV of generated passwords. Defaults to the current folder. |

---

## 4. Swagger (API docs)

- URL: `http://localhost:3000/api/docs`.
- Protected by **HTTP basic auth** using `SWAGGER_USER` and `SWAGGER_PASS` (see `src/main.ts`).
- To call protected routes, log in with `POST /v1/auth/login`, copy the access token, and click **Authorize** (`access-token` bearer scheme). Swagger remembers it (`persistAuthorization`).
- Admin routes are grouped by tags such as `Admin · Users`.

---

## 5. Database migrations

The schema is managed **only** by migrations in `src/migrations/`. `synchronize` is off. The CLI uses `src/config/database/data-source.ts`.

| Command | What it does |
| --- | --- |
| `npm run migration:run` | Apply all pending migrations. |
| `npm run migration:show` | List migrations and which are applied. |
| `npm run migration:revert` | Undo the last migration. |
| `npm run migration:generate -- src/migrations/<Name>` | Create a migration from the diff between entities and the DB. |
| `npm run migration:run:prod` (and `show:prod`, `revert:prod`) | Same, but from compiled `dist/`. Used in the Docker runtime image. |

!!! tip "Adding an entity"
    New entities live in `src/common/entities/` and must be added to **both** `src/config/database/data-source.ts` (for the CLI) and `src/config/database/database.module.ts` (for the app). Then generate a migration. Read the `typeorm-migrations` and `typeorm-entity-patterns` skills first.

---

## 6. Seeders and scripts

Run these after migrations, in this order:

| Command | What it does |
| --- | --- |
| `npm run seed` | Runs `src/seeder.ts`, which calls `DatabaseSeeder` (`src/seeders/index.ts`). It seeds roles, permissions, super-admin permissions, org role permissions, plan features, plans, the default AI model (from `OPENAI_MODEL`), and feature flags. |
| `npm run bootstrap-admin -- --email <email> --password <password>` | Creates the first `super_admin` user. Needs `npm run seed` first. Safe to run twice. |
| `npm run seed:assessment-templates` | Seeds assessment templates from `src/config/database/dummy-data/assessment-templates.ts`. |
| `npm run seed:roadmap-templates` | Seeds learning roadmap templates from `src/config/database/dummy-data/learning-roadmap-templates.ts`. |

The two template seeders accept `--owner-email <email>` (default owner is the `system` user), `--dry-run`, and `--help`. Some also accept `--update` to overwrite existing rows.

!!! note "Inside the runtime Docker image"
    The `ts-node` scripts do not work in the slim runtime image. Use `node dist/seeder.js` and `node dist/scripts/bootstrap-admin.js` there.

---

## 7. Tests

| Kind | Where | Command |
| --- | --- | --- |
| Unit | `tests/unit/<module>/...` mirroring `src/`, files named `*.spec.ts` | `npm run test` (watch: `npm run test:watch`) |
| Coverage | Collected from `src/**` into `coverage/` | `npm run test:cov` |
| E2E | `tests/*.e2e-spec.ts`, config in `tests/jest-e2e.json` | `npm run test:e2e` |
| Debug | Runs Jest in band with the inspector | `npm run test:debug` |

Unit tests mock repositories, `ConfigService`, and the LLM layer. See the `jest-mock-patterns`, `test-naming-conventions`, and `e2e-test-setup` skills in `.claude/skills/`.

---

## 8. Lint and format

| Command | What it does |
| --- | --- |
| `npm run lint` | ESLint over `src` and `tests`, **with `--fix`**. |
| `npm run format` | Prettier over `src/**/*.ts` and `tests/**/*.ts`. |
| `npm run build` | Compile to `dist/` with the Nest CLI. Good final check. |

A Husky pre-commit hook runs `npm run format`. CI (`.github/workflows/ci.yml`) runs lint, type checks, unit tests, build, migrations, and a container smoke test. CD (`cd.yml`) deploys the `dev` branch.

---

## 9. Useful tips

- **Code navigation.** The repo is indexed by CodeGraph. Run `npm run codegraph explore "<symbol or question>"` to see a symbol's source and its callers.
- **Project skills.** `.claude/skills/` holds the team's rules for folders, entities, migrations, validation, errors, guards, pagination, transactions, Swagger, and logging. Read the relevant one before you write code.
- **API-only instance.** Set `CONSUMER=false` to run an instance that publishes jobs but does not consume them.
- **Watch the queues.** The RabbitMQ UI at `http://localhost:15672` shows each queue and its message count. See [Architecture](architecture.md#rabbitmq) for the full list.
