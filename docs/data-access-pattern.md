---
title: Data Access Patterns
description: SQL query patterns for the TestFlow backend, matched to the current schema (see erd.md)
status: stable
last_updated: 2026-07-15
---

# Data Access Patterns

Query patterns below reflect the schema actually in place (see [ERD](erd.md)) — the previous version of this page described tables (`tests`, `test_participants`, `submissions`) that don't exist in this codebase; the real names are `assessments`, `assessments_participants`, `assessment_attempts`, `assessment_attempt_responses`.

All soft-deletable tables use `deleted_at IS NULL` to exclude deleted rows — include this in every query unless intentionally including deleted rows (e.g. admin restore tooling).

## 👤 Users & Auth

| Scenario | DQL | Remark |
| --- | --- | --- |
| Get user by email | `SELECT * FROM users WHERE email = 'john@example.com' AND deleted_at IS NULL;` | |
| Get user by username | `SELECT * FROM users WHERE username = 'john123' AND deleted_at IS NULL;` | |
| Login lookup (email OR username) | `SELECT * FROM users WHERE (email = $1 OR username = $1) AND deleted_at IS NULL;` | `AuthService.login` resolves by whichever field the client supplied |
| Check email/username/phone taken | `SELECT * FROM users WHERE (email = $1 OR username = $2 OR phone_number = $3) AND deleted_at IS NULL;` | `UsersService.create` runs one combined existence check, not three separate ones |
| Get user by id (post-JWT-verify) | `SELECT * FROM users WHERE id = $1 AND deleted_at IS NULL;` | |
| Store refresh token (upsert-by-user) | `INSERT INTO refresh_tokens (token_hash, user_id, expires_at) VALUES ($1,$2,$3) ON CONFLICT ...` conceptually — actual code does a `findOne` by `user_id` (`withDeleted: true`) then either updates in place (clearing `deleted_at`) or inserts | One active refresh-token row per user, not per device |
| Look up refresh token on rotation | `SELECT * FROM refresh_tokens WHERE user_id = $1 AND token_hash = $2 AND deleted_at IS NULL;` | Token hash is SHA-256 of the raw JWT — raw tokens are never stored |
| Revoke refresh token (rotation or logout) | `UPDATE refresh_tokens SET deleted_at = NOW() WHERE token_hash = $1;` | Soft delete, not hard delete |
| Get all users created in last 7 days | `SELECT * FROM users WHERE created_at >= NOW() - INTERVAL '7 DAYS' AND deleted_at IS NULL ORDER BY created_at DESC;` | Not currently exposed via any endpoint — `UsersService.findAll()` exists but has no controller route |

## 🔐 RBAC

| Scenario | DQL | Remark |
| --- | --- | --- |
| Get all roles | `SELECT * FROM roles WHERE deleted_at IS NULL;` | |
| Get all resources | `SELECT * FROM resources WHERE deleted_at IS NULL;` | |
| Get all resource keys granted to a role | `SELECT res.key FROM role_permissions rp JOIN roles r ON r.id = rp.role_id JOIN resources res ON res.id = rp.resource_id WHERE r.key = 'super_admin' AND rp.deleted_at IS NULL;` | |
| Get a user's role keys (used by RolesGuard) | `SELECT r.key FROM users_roles ur JOIN roles r ON r.id = ur.role_id WHERE ur.user_id = $1 AND ur.deleted_at IS NULL AND r.deleted_at IS NULL;` | |
| Check permission (with wildcard fallback — used by PermissionGuard) | `SELECT 1 FROM users_roles ur JOIN roles r ON r.id = ur.role_id JOIN role_permissions rp ON rp.role_id = r.id JOIN resources res ON res.id = rp.resource_id WHERE ur.user_id = $1 AND (res.key = $2 OR res.key = '*:*') LIMIT 1;` | This is the actual runtime authorization check — note the `OR res.key = '*:*'` wildcard branch |
| Strict access check (admin tooling, no wildcard) | `SELECT 1 FROM role_permissions rp JOIN roles r ON r.id = rp.role_id JOIN resources res ON res.id = rp.resource_id WHERE r.key = $1 AND res.key = $2 AND rp.deleted_at IS NULL;` | Backs `POST /v1/access-check` — deliberately does not apply the wildcard fallback |
| Assign role to user | `INSERT INTO users_roles (user_id, role_id) VALUES ($1, $2);` | 409 if an active row already exists for this (user_id, role_id) |
| Find roles without any granted permissions | `SELECT r.* FROM roles r LEFT JOIN role_permissions rp ON rp.role_id = r.id AND rp.deleted_at IS NULL WHERE rp.id IS NULL AND r.deleted_at IS NULL;` | Useful for spotting a misconfigured role |
| Full permission matrix (admin panel) | `SELECT r.key AS role_key, res.key AS resource_key, (rp.id IS NOT NULL) AS has_access FROM roles r CROSS JOIN resources res LEFT JOIN role_permissions rp ON rp.role_id = r.id AND rp.resource_id = res.id AND rp.deleted_at IS NULL ORDER BY r.key, res.key;` | |

## ❓ Questions

| Scenario | DQL | Remark |
| --- | --- | --- |
| Create question | `INSERT INTO questions (question_text, options, correct_option_id, created_by) VALUES ($1, $2::jsonb, $3, $4);` | `options` is a jsonb array of `{id, text}` |
| Get question by id | `SELECT * FROM questions WHERE id = $1 AND deleted_at IS NULL;` | |
| Search questions by text | `SELECT * FROM questions WHERE question_text ILIKE '%' || $1 || '%' AND deleted_at IS NULL ORDER BY created_at DESC;` | Backs `GET /v1/questions?search=` |
| Get questions created by a user | `SELECT * FROM questions WHERE created_by = $1 AND deleted_at IS NULL ORDER BY created_at DESC;` | |
| Pull N random questions (assessment "generate") | `SELECT * FROM questions WHERE deleted_at IS NULL ORDER BY RANDOM() LIMIT $1;` | Backs `POST /v1/assessments/generate` — this is the entire "AI generation" today; there is no LLM call |
| Soft delete question | `UPDATE questions SET deleted_at = NOW() WHERE id = $1;` | |

## 📝 Assessments

| Scenario | DQL | Remark |
| --- | --- | --- |
| Create assessment shell | `INSERT INTO assessments (description, start_time, end_time, duration, max_participants, max_total_questions, correct_marks, incorrect_marks, created_by) VALUES (...);` | `current_participants`/`curr_total_questions` default to 0 |
| Get assessment by id | `SELECT * FROM assessments WHERE id = $1 AND deleted_at IS NULL;` | |
| Get upcoming/ongoing/completed assessments | `SELECT * FROM assessments WHERE start_time > NOW() AND deleted_at IS NULL;` / `WHERE start_time <= NOW() AND end_time >= NOW();` / `WHERE end_time < NOW();` | Status is derived at query time (`AssessmentStatus` enum), never stored as a column |
| Get assessments created by a user | `SELECT * FROM assessments WHERE created_by = $1 AND deleted_at IS NULL ORDER BY created_at DESC;` | Backs `GET /v1/assessments/me` |
| Search assessments by description | `SELECT * FROM assessments WHERE description ILIKE '%' || $1 || '%' AND deleted_at IS NULL;` | |
| Get assessments by popularity | `SELECT a.*, COUNT(ap.id) AS participant_count FROM assessments a LEFT JOIN assessments_participants ap ON ap.assessment_id = a.id AND ap.deleted_at IS NULL WHERE a.deleted_at IS NULL GROUP BY a.id ORDER BY participant_count DESC;` | Backs `?popular=true` |
| Soft delete assessment (creator-only) | `UPDATE assessments SET deleted_at = NOW() WHERE id = $1 AND created_by = $2;` | Ownership check happens in the service before this runs, not via a WHERE-clause-only guard |

## 🔗 Assessment Questions (snapshot mapping)

| Scenario | DQL | Remark |
| --- | --- | --- |
| Add question to assessment (snapshot) | `INSERT INTO assessments_questions (assessment_id, question_id, question_text, options, correct_option_id, correct_marks, incorrect_marks, order_index) SELECT $1, q.id, q.question_text, q.options, q.correct_option_id, $2, $3, $4 FROM questions q WHERE q.id = $5;` | Denormalizes question content at add-time — later edits to `questions` do not retroactively change already-assigned assessment questions |
| Get all questions for an assessment, ordered | `SELECT * FROM assessments_questions WHERE assessment_id = $1 ORDER BY order_index ASC;` | |
| Get total marks available for an assessment | `SELECT SUM(correct_marks) AS max_marks FROM assessments_questions WHERE assessment_id = $1;` | |
| Check for duplicate question in assessment | `SELECT 1 FROM assessments_questions WHERE assessment_id = $1 AND question_id = $2;` | DB also enforces this via a unique index on `(assessment_id, question_id)` |
| Remaining capacity check before adding questions | `SELECT max_total_questions - curr_total_questions AS remaining FROM assessments WHERE id = $1;` | Compared against the incoming batch size in `AssessmentsService.addQuestions` |

## 👥 Assessment Participants

| Scenario | DQL | Remark |
| --- | --- | --- |
| Register (enroll) | `INSERT INTO assessments_participants (user_id, assessment_id, status) VALUES ($1, $2, 'not_started'); UPDATE assessments SET current_participants = current_participants + 1 WHERE id = $2;` | Both statements run in one transaction |
| Check enrollment | `SELECT * FROM assessments_participants WHERE user_id = $1 AND assessment_id = $2 AND deleted_at IS NULL;` | Backs `POST /v1/assessments/:id/check-enrollment` |
| Unregister | `UPDATE assessments_participants SET deleted_at = NOW() WHERE user_id = $1 AND assessment_id = $2; UPDATE assessments SET current_participants = current_participants - 1 WHERE id = $2;` | |
| Mark participant completed (on submit) | `UPDATE assessments_participants SET status = 'completed', completed_at = NOW() WHERE user_id = $1 AND assessment_id = $2;` | Called from `AttemptsService.submitAttempt()` via `AssessmentsService.markParticipantCompleted()` |
| Get all participants of an assessment | `SELECT u.id, u.full_name, u.email, ap.status FROM assessments_participants ap JOIN users u ON u.id = ap.user_id WHERE ap.assessment_id = $1 AND ap.deleted_at IS NULL;` | |
| Count participants per assessment | `SELECT assessment_id, COUNT(*) FROM assessments_participants WHERE deleted_at IS NULL GROUP BY assessment_id;` | Note: `current_participants` on `assessments` is a denormalized counter maintained by the app, not derived by this query at read time |

## 🎯 Attempts & Responses

| Scenario | DQL | Remark |
| --- | --- | --- |
| Start attempt (check existing first) | `SELECT * FROM assessment_attempts WHERE user_id = $1 AND assessment_id = $2;` | If found, its `uid` is returned as-is (resume); DB also enforces one-per-user-per-assessment via a unique constraint |
| Create attempt | `INSERT INTO assessment_attempts (assessment_id, user_id, started_at, total_questions, status) VALUES ($1, $2, NOW(), $3, 'IN_PROGRESS') RETURNING uid;` | `total_questions` is snapshotted from `assessments.curr_total_questions` at start time |
| Get attempt by uid (ownership-checked) | `SELECT * FROM assessment_attempts WHERE uid = $1;` | `AttemptsService.resolveAttempt()` then compares `user_id` to the caller in application code (403 if mismatched), not in the WHERE clause |
| Upsert a response | `SELECT * FROM assessment_attempt_responses WHERE attempt_id = $1 AND assessment_question_id = $2;` then `UPDATE ... SET selected_option_id, is_correct, marks_awarded, answered_at = NOW()` or `INSERT ...` | `is_correct`/`marks_awarded` are computed in application code by comparing `optionId` to `assessments_questions.correct_option_id`, then applying `correct_marks`/`incorrect_marks` |
| Delete a response | `DELETE FROM assessment_attempt_responses WHERE attempt_id = $1 AND assessment_question_id = $2;` | Only allowed while `attempt.status = 'IN_PROGRESS'` and the assessment window is still open |
| Submit attempt (guarded against double-submit) | `UPDATE assessment_attempts SET status = 'SUBMITTED', submitted_at = NOW() WHERE id = $1 AND status = 'IN_PROGRESS';` | Checks `affected = 0` to detect an already-submitted attempt and reject with 400 |
| Evaluate (aggregate responses → attempt) | `SELECT COUNT(*) AS attempted, COUNT(*) FILTER (WHERE is_correct) AS correct, COUNT(*) FILTER (WHERE NOT is_correct) AS incorrect, SUM(marks_awarded) AS score FROM assessment_attempt_responses WHERE attempt_id = $1;` then `UPDATE assessment_attempts SET score=$2, attempted_questions=$3, correct_responses=$4, incorrect_responses=$5, status='EVALUATED' WHERE id = $1;` | Runs async/unawaited right after submit, inside `EvaluationService.evaluate()` |
| Get evaluated result with per-question breakdown | `SELECT aq.*, r.selected_option_id, r.is_correct, r.marks_awarded FROM assessments_questions aq LEFT JOIN assessment_attempt_responses r ON r.assessment_question_id = aq.id AND r.attempt_id = $1 WHERE aq.assessment_id = $2 ORDER BY aq.order_index;` | Rejected with 400 unless the attempt's `status = 'EVALUATED'` |
| List a user's attempts, filterable by status | `SELECT * FROM assessment_attempts WHERE user_id = $1 AND ($2::text IS NULL OR status = $2) ORDER BY created_at DESC;` | Backs `GET /v1/attempts/me` |

## 💳 Commerce — Plans & Versions

| Scenario | DQL | Remark |
| --- | --- | --- |
| List active plans with current version | `SELECT p.*, pv.* FROM plans p JOIN plan_versions pv ON pv.plan_id = p.id AND pv.is_current = true WHERE p.is_active = true AND p.deleted_at IS NULL;` | Backs `GET /v1/plans` — historical (non-current) versions are never returned here |
| Resolve plan + version at checkout | `SELECT * FROM plans WHERE code = $1 AND deleted_at IS NULL;` then `SELECT * FROM plan_versions WHERE plan_id = $2 AND version = $3 AND deleted_at IS NULL;` | Two sequential lookups, both required before a checkout can proceed |
| Check version still purchasable | `SELECT (effective_until IS NULL OR effective_until >= NOW()) AS purchasable FROM plan_versions WHERE id = $1;` | |
| Demote current version, then insert new one | `UPDATE plan_versions SET is_current = false, effective_until = COALESCE(effective_until, $2) WHERE plan_id = $1 AND is_current = true;` then `INSERT INTO plan_versions (plan_id, version, is_current, ...) VALUES ($1, $3, true, ...);` | Both run in one transaction — the partial unique index on `(plan_id) WHERE is_current = true` is what makes this safe under concurrent writes |
| Get features for a plan version | `SELECT * FROM plan_features WHERE plan_version_id = $1 AND deleted_at IS NULL;` | Table is `plan_features`, despite the TypeORM entity class being named `PlanVersionFeature` |

## 💰 Commerce — Subscriptions & Checkout

| Scenario | DQL | Remark |
| --- | --- | --- |
| Check for an existing active subscription to a plan | `SELECT 1 FROM subscriptions s JOIN plan_versions pv ON pv.id = s.plan_version_id WHERE s.user_id = $1 AND s.status = 'ACTIVE' AND pv.plan_id = $2;` | Only blocks on `ACTIVE` — a `PENDING` subscription for the same plan doesn't block a second checkout |
| Create pending subscription | `INSERT INTO subscriptions (user_id, plan_version_id, status, starts_at, expires_at, auto_renew) VALUES ($1, $2, 'PENDING', NOW(), $3, true);` | `expires_at` = now + 1 month or +1 year depending on `plan_versions.billing_cycle` |
| Create pending order for the subscription | `INSERT INTO orders (user_id, resource_type, resource_id, total_amount, currency, provider, provider_order_id, status, expires_at) VALUES ($1, 'SUBSCRIPTION', $2, $3, $4, 'RAZORPAY', $5, 'PENDING', $6);` | `provider_order_id` is a placeholder (`chk_<uuid>`) until the real Razorpay order-create call happens |
| Activate subscription on payment capture | `UPDATE subscriptions SET status = 'ACTIVE' WHERE id = $1;` | Executed directly from `PaymentsService`, not `SubscriptionsService` — a documented cross-module shortcut (see [ERD notes](erd.md#3-commerce--plans-subscriptions--usage)) |
| Get a user's subscription history | `SELECT s.*, pv.version, p.code, p.name FROM subscriptions s JOIN plan_versions pv ON pv.id = s.plan_version_id JOIN plans p ON p.id = pv.plan_id WHERE s.user_id = $1 ORDER BY s.created_at DESC;` | Not currently exposed via any endpoint |

## 🧾 Payments & Orders

| Scenario | DQL | Remark |
| --- | --- | --- |
| Load order for payment creation (ownership + status checks) | `SELECT * FROM orders WHERE id = $1;` | Service then checks `user_id = caller` (403), `status = 'PENDING'` (422 if not), `expires_at > NOW()` (422 if expired) |
| Reuse an existing pending payment (idempotent create) | `SELECT * FROM payments WHERE order_id = $1 ORDER BY created_at DESC LIMIT 1;` | If `status = 'SUCCESS'` → 409; if `PENDING` with a `provider_order_id` set → reused as-is instead of calling Razorpay again |
| Create payment row after Razorpay order-create | `INSERT INTO payments (order_id, amount, currency, provider, provider_order_id, status) VALUES ($1, $2, $3, 'RAZORPAY', $4, 'PENDING');` | |
| Mark payment captured (webhook-driven) | `SELECT * FROM payments WHERE provider_order_id = $1;` then, in one transaction: `UPDATE payments SET status='SUCCESS', provider_payment_id=$2, paid_at=$3, failure_reason=NULL WHERE id=$4;` `UPDATE orders SET status='PAID' WHERE id=$5;` and conditionally the subscription-activation update above | No-ops silently (logs a warning) if no matching payment is found — webhook still returns 200 |
| Mark payment failed | `UPDATE payments SET status='FAILED', provider_payment_id=$1, failure_reason=$2 WHERE id=$3;` | Does **not** touch `orders` or `subscriptions` — order remains `PENDING`, allowing retry until `expires_at` |
| Webhook idempotency check | `SELECT processed FROM webhook_events WHERE event_id = $1;` | `event_id` is synthesized as `"{event}:{razorpayPaymentId}"`, not Razorpay's own webhook delivery ID |
| Record webhook processed | `INSERT INTO webhook_events (provider, event_id, event_type, payload, processed, processed_at) VALUES ('RAZORPAY', $1, $2, $3::jsonb, true, NOW()) ON CONFLICT (event_id) DO UPDATE SET processed = true, processed_at = NOW();` | Conceptual — actual code does a find-then-update-or-insert, not a literal `ON CONFLICT` |

## 📊 Usage Tracking

> **Status: schema only, not implemented.** `user_feature_usage` and `plan_features.reset_period` exist and are ready to support quota tracking, but `UsageService` currently has zero methods. The query patterns below are the intended shape once it's built — not yet real code paths.

| Scenario | DQL | Remark |
| --- | --- | --- |
| Get current usage for a feature | `SELECT * FROM user_feature_usage WHERE user_id = $1 AND plan_feature_id = $2 AND period_end >= CURRENT_DATE;` | Not yet called anywhere |
| Increment usage | `UPDATE user_feature_usage SET used = used + 1 WHERE user_id = $1 AND plan_feature_id = $2 AND period_end >= CURRENT_DATE;` | Not yet called anywhere |
| Reset usage for a new period | `INSERT INTO user_feature_usage (user_id, plan_feature_id, used, period_start, period_end) VALUES ($1, $2, 0, $3, $3 + ($4 || ' days')::interval);` | `$4` = `plan_features.reset_period`; no scheduler/cron exists yet to trigger this |

## 🧹 Integrity / Validation Queries

| Scenario | DQL | Remark |
| --- | --- | --- |
| Find questions never used in any assessment | `SELECT q.* FROM questions q LEFT JOIN assessments_questions aq ON aq.question_id = q.id WHERE aq.id IS NULL AND q.deleted_at IS NULL;` | |
| Find assessments with zero questions | `SELECT * FROM assessments WHERE curr_total_questions = 0 AND deleted_at IS NULL;` | |
| Find attempts stuck `IN_PROGRESS` past the assessment window | `SELECT aa.* FROM assessment_attempts aa JOIN assessments a ON a.id = aa.assessment_id WHERE aa.status = 'IN_PROGRESS' AND a.end_time < NOW();` | No auto-submit job currently exists to clean these up |
| Find orders past `expires_at` still `PENDING` | `SELECT * FROM orders WHERE status = 'PENDING' AND expires_at < NOW();` | No cleanup/expiry job currently marks these `FAILED` |
| Find users without any role | `SELECT u.* FROM users u LEFT JOIN users_roles ur ON ur.user_id = u.id AND ur.deleted_at IS NULL WHERE ur.id IS NULL AND u.deleted_at IS NULL;` | Every non-admin user is expected to have at least one role assigned via `POST /v1/users/:username/roles` |
