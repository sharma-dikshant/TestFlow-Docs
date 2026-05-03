# Data Access Patterns

## 👤 Users & Auth

### Important Note:

- We use soft delete by having a `deleted_at` timestamp column. All queries should include `AND deleted_at IS NULL` to exclude soft-deleted records. `deleted_at` is used by admin to soft delete user accounts, allowing for potential restoration later. This approach ensures that deleted accounts are not permanently removed from the database, providing flexibility for account recovery if needed.
- We supporting account activation/inactivation using a boolean flag `is_active`. Queries should include `AND is_active = true` to only consider active accounts.

| Scenario                             | DQL                                                                                                                                                                                                        | Remark                                                 |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| Get user by email                    | `SELECT * FROM users WHERE email = 'john@example.com' AND deleted_at IS NULL`                                                                                                                              |                                                        |
| Get user by username                 | `SELECT * FROM users WHERE username = 'john123' AND deleted_at IS NULL AND is_active = true`                                                                                                               |                                                        |
| Login user                           | `SELECT * FROM users WHERE (email = 'john@example.com' OR username = 'john123') AND deleted_at IS NULL AND is_active = true;`                                                                              |                                                        |
| Check if email already exists        | `SELECT EXISTS (SELECT 1 FROM users WHERE email = 'john@example.com' AND deleted_at IS NULL);`                                                                                                             |                                                        |
| Check if username already exists     | `SELECT EXISTS (SELECT 1 FROM users WHERE username = 'john123' AND deleted_at IS NULL);`                                                                                                                   |                                                        |
| Get all active users                 | `SELECT * FROM users WHERE is_active = true AND deleted_at IS NULL;`                                                                                                                                       | Instead of enum status we use a boolean flag is_active |
| Get all verified users               | `SELECT * FROM users WHERE is_verified = true AND deleted_at IS NULL AND is_active = true;`                                                                                                                |                                                        |
| Get all users by role                | `SELECT u.* FROM users u JOIN user_roles ur ON ur.user_id = u.id JOIN roles r ON r.id = ur.role_id WHERE r.name = 'ADMIN' AND u.deleted_at IS NULL AND u.is_active = true;`                                |                                                        |
| Get user profile with role           | `SELECT u.id, u.full_name, u.email, r.name AS role FROM users u JOIN user_roles ur ON ur.user_id = u.id JOIN roles r ON r.id = ur.role_id WHERE u.id = 1 AND u.deleted_at IS NULL AND u.is_active = true;` |                                                        |
| Get recently registered users        | `SELECT * FROM users WHERE created_at >= NOW() - INTERVAL '24 HOURS' AND deleted_at IS NULL ORDER BY created_at DESC;`                                                                                     |                                                        |
| Soft delete user                     | `UPDATE users SET deleted_at = NOW() WHERE id = 1 AND deleted_at IS NULL AND is_active = true;`                                                                                                            |                                                        |
| Restore deleted user                 | `UPDATE users SET deleted_at = NULL WHERE id = 1 AND deleted_at IS NOT NULL;`                                                                                                                              |                                                        |
| Inactive User account                | `UPDATE users SET is_active = false WHERE username = 'john123' AND deleted_at IS NULL AND is_active = true;`                                                                                               |                                                        |
| Active User account                  | `UPDATE users SET is_active = true WHERE username = 'john123' AND deleted_at IS NULL AND is_active = false;`                                                                                               |                                                        |
| Search users by name/email           | `SELECT * FROM users WHERE (full_name ILIKE '%john%' OR email ILIKE '%john%') AND deleted_at IS NULL AND is_active = true;`                                                                                |                                                        |
| Get all users created in last 7 days | `SELECT * FROM users WHERE created_at >= NOW() - INTERVAL '7 DAYS' AND deleted_at IS NULL ORDER BY created_at DESC;`                                                                                       |                                                        |

## 🔐 RBAC / Authorization

| Scenario                                      | DQL                                                                                                                                                                                                                                                          | Remark |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------ |
| Get all roles                                 | `SELECT * FROM roles WHERE deleted_at IS NULL;`                                                                                                                                                                                                              |        |
| Get all resources                             | `SELECT * FROM resources WHERE deleted_at IS NULL;`                                                                                                                                                                                                          |        |
| Get all policies for a role                   | `SELECT p.* FROM policies p JOIN roles r ON r.id = p.role_id WHERE r.name = 'ADMIN' AND p.deleted_at IS NULL;`                                                                                                                                               |        |
| Get all resources accessible by a role        | `SELECT res.* FROM resources res JOIN policies p ON p.resource_id = res.id JOIN roles r ON r.id = p.role_id WHERE r.name = 'ADMIN' AND res.deleted_at IS NULL;`                                                                                              |        |
| Check whether role has access to resource     | `SELECT EXISTS ( SELECT 1 FROM policies p JOIN roles r ON r.id = p.role_id JOIN resources res ON res.id = p.resource_id WHERE r.name = 'ADMIN' AND res.key = 'tests:create' AND p.deleted_at IS NULL );`                                                     |        |
| Get all users having a specific role          | `SELECT u.* FROM users u JOIN user_roles ur ON ur.user_id = u.id JOIN roles r ON r.id = ur.role_id WHERE r.name = 'ADMIN' AND u.deleted_at IS NULL;`                                                                                                         |        |
| Get all roles using a specific resource       | `SELECT r.* FROM roles r JOIN policies p ON p.role_id = r.id JOIN resources res ON res.id = p.resource_id WHERE res.key = 'tests:create' AND r.deleted_at IS NULL;`                                                                                          |        |
| Get all policies with role + resource details | `SELECT p.id, r.name AS role_name, res.key AS resource_key, p.description FROM policies p JOIN roles r ON r.id = p.role_id JOIN resources res ON res.id = p.resource_id WHERE p.deleted_at IS NULL;`                                                         |        |
| Find unused resources                         | `SELECT res.* FROM resources res LEFT JOIN policies p ON p.resource_id = res.id WHERE p.id IS NULL AND res.deleted_at IS NULL;`                                                                                                                              |        |
| Find roles without policies                   | `SELECT r.* FROM roles r LEFT JOIN policies p ON p.role_id = r.id WHERE p.id IS NULL AND r.deleted_at IS NULL;`                                                                                                                                              |        |
| Get all permissions matrix for admin panel    | `SELECT r.name AS role_name, res.key AS resource_key, CASE WHEN p.id IS NOT NULL THEN true ELSE false END AS has_access FROM roles r CROSS JOIN resources res LEFT JOIN policies p ON p.role_id = r.id AND p.resource_id = res.id ORDER BY r.name, res.key;` |        |

## 📝 Tests

| Scenario                               | DQL                                                                                                                                                                                                                    | Remark |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Create test                            | `INSERT INTO tests (title, description, start_time, end_time, duration, created_by, created_at, updated_at) VALUES ('JavaScript Test', 'Basic JS assessment', NOW(), NOW() + INTERVAL '1 HOUR', 60, 1, NOW(), NOW());` |        |
| Get test by id                         | `SELECT * FROM tests WHERE id = 1;`                                                                                                                                                                                    |        |
| Get all tests                          | `SELECT * FROM tests ORDER BY created_at DESC;`                                                                                                                                                                        |        |
| Get all upcoming tests                 | `SELECT * FROM tests WHERE start_time > NOW() ORDER BY start_time ASC;`                                                                                                                                                |        |
| Get all ongoing tests                  | `SELECT * FROM tests WHERE start_time <= NOW() AND end_time >= NOW();`                                                                                                                                                 |        |
| Get all completed tests                | `SELECT * FROM tests WHERE end_time < NOW();`                                                                                                                                                                          |        |
| Get all tests created by a user        | `SELECT * FROM tests WHERE created_by = 1 ORDER BY created_at DESC;`                                                                                                                                                   |        |
| Search tests by title                  | `SELECT * FROM tests WHERE title ILIKE '%javascript%';`                                                                                                                                                                |        |
| Get test with creator details          | `SELECT t.*, u.id AS creator_id, u.full_name, u.email FROM tests t JOIN users u ON u.id = t.created_by WHERE t.id = 1;`                                                                                                |        |
| Get tests between dates                | `SELECT * FROM tests WHERE start_time BETWEEN '2026-05-01' AND '2026-05-31';`                                                                                                                                          |        |
| Get tests with participant count       | `SELECT t.id, t.title, COUNT(tp.id) AS participant_count FROM tests t LEFT JOIN test_participants tp ON tp.test_id = t.id GROUP BY t.id, t.title;`                                                                     |        |
| Get tests with question count          | `SELECT t.id, t.title, COUNT(tq.id) AS question_count FROM tests t LEFT JOIN test_questions tq ON tq.test_id = t.id GROUP BY t.id, t.title;`                                                                           |        |
| Get tests sorted by latest             | `SELECT * FROM tests ORDER BY created_at DESC;`                                                                                                                                                                        |        |
| Get tests sorted by popularity         | `SELECT t.id, t.title, COUNT(tp.id) AS participant_count FROM tests t LEFT JOIN test_participants tp ON tp.test_id = t.id GROUP BY t.id, t.title ORDER BY participant_count DESC;`                                     |        |
| Soft delete test                       | `UPDATE tests SET deleted_at = NOW() WHERE id = 1;`                                                                                                                                                                    |        |
| Get tests available for a user         | `SELECT * FROM tests WHERE start_time > NOW() AND id NOT IN ( SELECT test_id FROM test_participants WHERE user_id = 1 );`                                                                                              |        |
| Get tests user already participated in | `SELECT t.* FROM tests t JOIN test_participants tp ON tp.test_id = t.id WHERE tp.user_id = 1;`                                                                                                                         |        |
| Get currently active tests             | `SELECT * FROM tests WHERE start_time <= NOW() AND end_time >= NOW();`                                                                                                                                                 |        |
| Get tests ending soon                  | `SELECT * FROM tests WHERE end_time BETWEEN NOW() AND NOW() + INTERVAL '30 MINUTES';`                                                                                                                                  |        |

## ❓ Questions

| Scenario                             | DQL | Remark |
| ------------------------------------ | --- | ------ |
| Create question                      | `INSERT INTO questions (question_text, type, options, test_cases, default_marks, created_by, created_at, updated_at) VALUES ('What is 2+2?', 'MCQ', '[{"text":"3","isCorrect":false},{"text":"4","isCorrect":true}]'::jsonb, NULL, 1, 1, NOW(), NOW());` | MCQ → fill `options`; CODING → fill `test_cases`; SUBJECTIVE → both NULL |
| Get question by id                   | `SELECT * FROM questions WHERE id = 1;` |        |
| Get all questions                    | `SELECT * FROM questions ORDER BY created_at DESC;` |        |
| Get questions by type                | `SELECT * FROM questions WHERE type = 'MCQ';` |        |
| Get questions created by a user      | `SELECT * FROM questions WHERE created_by = 1 ORDER BY created_at DESC;` |        |
| Search questions by text             | `SELECT * FROM questions WHERE question_text ILIKE '%binary search%';` |        |
| Get questions with default marks     | `SELECT * FROM questions WHERE default_marks = 5;` |        |
| Get questions used in tests          | `SELECT DISTINCT q.* FROM questions q JOIN test_questions tq ON tq.question_id = q.id;` |        |
| Get unused questions                 | `SELECT * FROM questions WHERE id NOT IN (SELECT question_id FROM test_questions);` |        |
| Get most frequently used questions   | `SELECT q.id, q.question_text, COUNT(tq.id) AS usage_count FROM questions q JOIN test_questions tq ON tq.question_id = q.id GROUP BY q.id, q.question_text ORDER BY usage_count DESC LIMIT 10;` |        |
| Get coding questions only            | `SELECT * FROM questions WHERE test_cases IS NOT NULL AND jsonb_array_length(test_cases) > 0;` |        |
| Get MCQ questions only               | `SELECT * FROM questions WHERE options IS NOT NULL AND jsonb_array_length(options) > 0;` |        |
| Get subjective questions only        | `SELECT * FROM questions WHERE (options IS NULL OR jsonb_array_length(options) = 0) AND (test_cases IS NULL OR jsonb_array_length(test_cases) = 0);` |        |
| Get question with all tests using it | `SELECT t.id, t.title FROM tests t JOIN test_questions tq ON tq.test_id = t.id WHERE tq.question_id = 1;` |        |
| Get recently created questions       | `SELECT * FROM questions ORDER BY created_at DESC LIMIT 10;` |        |

## 🔗 Test Questions Mapping

| Scenario                            | DQL | Remark |
| ----------------------------------- | --- | ------ |
| Add question to test                | `INSERT INTO test_questions (test_id, question_id, marks, order_index) VALUES (1, 10, 5, 1);` | `marks` overrides `questions.default_marks` for this test |
| Get all questions for a test        | `SELECT q.*, tq.marks, tq.order_index FROM test_questions tq JOIN questions q ON q.id = tq.question_id WHERE tq.test_id = 1;` |        |
| Get ordered questions for a test    | `SELECT q.*, tq.marks, tq.order_index FROM test_questions tq JOIN questions q ON q.id = tq.question_id WHERE tq.test_id = 1 ORDER BY tq.order_index ASC;` |        |
| Get total marks of a test           | `SELECT SUM(marks) AS total_marks FROM test_questions WHERE test_id = 1;` |        |
| Get question count for a test       | `SELECT COUNT(*) AS question_count FROM test_questions WHERE test_id = 1;` |        |
| Get all tests containing a question | `SELECT t.* FROM tests t JOIN test_questions tq ON tq.test_id = t.id WHERE tq.question_id = 10;` |        |
| Update question order in test       | `UPDATE test_questions SET order_index = 3 WHERE test_id = 1 AND question_id = 10;` |        |
| Update marks override in test       | `UPDATE test_questions SET marks = 10 WHERE test_id = 1 AND question_id = 10;` |        |
| Remove question from test           | `DELETE FROM test_questions WHERE test_id = 1 AND question_id = 10;` |        |
| Find duplicate question mappings    | `SELECT test_id, question_id, COUNT(*) FROM test_questions GROUP BY test_id, question_id HAVING COUNT(*) > 1;` |        |
| Get first/last question of a test   | `SELECT * FROM test_questions WHERE test_id = 1 AND order_index IN ((SELECT MIN(order_index) FROM test_questions WHERE test_id = 1), (SELECT MAX(order_index) FROM test_questions WHERE test_id = 1));` |        |
| Get questions with custom marks     | `SELECT q.id, q.default_marks, tq.marks, tq.test_id FROM test_questions tq JOIN questions q ON q.id = tq.question_id WHERE tq.marks IS DISTINCT FROM q.default_marks;` | Rows where the per-test override differs from `default_marks` |

## 👥 Test Participants

| Scenario                               | DQL | Remark |
| -------------------------------------- | --- | ------ |
| Enroll user in test                    | `INSERT INTO test_participants (user_id, test_id, status, created_at) VALUES (1, 1, 'ENROLLED', NOW());` |        |
| Get all participants of a test         | `SELECT u.id, u.full_name, u.email, tp.status FROM test_participants tp JOIN users u ON u.id = tp.user_id WHERE tp.test_id = 1;` |        |
| Get all tests participated by a user   | `SELECT t.* FROM tests t JOIN test_participants tp ON tp.test_id = t.id WHERE tp.user_id = 1;` |        |
| Get participant status                 | `SELECT id AS participant_id, status FROM test_participants WHERE user_id = 1 AND test_id = 1 ORDER BY created_at;` | One row per attempt |
| Get participants currently taking test | `SELECT * FROM test_participants WHERE test_id = 1 AND status = 'IN_PROGRESS';` |        |
| Get completed participants             | `SELECT * FROM test_participants WHERE test_id = 1 AND status = 'COMPLETED';` |        |
| Get absent participants                | `SELECT * FROM test_participants WHERE test_id = 1 AND started_at IS NULL;` | Enrolled but never started |
| Get participant start/completion time  | `SELECT id, started_at, completed_at FROM test_participants WHERE id = 1;` |        |
| Get average completion time            | `SELECT AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) AS avg_seconds FROM test_participants WHERE test_id = 1 AND completed_at IS NOT NULL;` |        |
| Count participants per test            | `SELECT test_id, COUNT(*) AS participant_count FROM test_participants GROUP BY test_id;` |        |
| Count tests attempted by user          | `SELECT user_id, COUNT(DISTINCT test_id) AS tests_attempted FROM test_participants WHERE user_id = 1 GROUP BY user_id;` |        |
| Get leaderboard participation list     | `SELECT tp.id AS participant_id, u.full_name, COALESCE(SUM(s.score), 0) AS total_score FROM test_participants tp JOIN users u ON u.id = tp.user_id LEFT JOIN submissions s ON s.participant_id = tp.id WHERE tp.test_id = 1 GROUP BY tp.id, u.full_name ORDER BY total_score DESC;` |        |
| Get latest participants                | `SELECT * FROM test_participants ORDER BY created_at DESC LIMIT 10;` |        |
| Check if user already joined test      | `SELECT id AS participant_id, status, started_at, completed_at, created_at FROM test_participants WHERE user_id = 1 AND test_id = 1 ORDER BY created_at;` | One row per attempt; user may have multiple |
| Get participant by user + test         | `SELECT * FROM test_participants WHERE user_id = 1 AND test_id = 1 ORDER BY created_at DESC;` | Returns one row per attempt; segregated by `participant_id` (use `LIMIT 1` for the latest attempt) |
| Get participants who never submitted   | `SELECT tp.* FROM test_participants tp LEFT JOIN submissions s ON s.participant_id = tp.id WHERE s.id IS NULL;` | Per attempt — a `participant_id` with no submissions |

## 📤 Submissions

| Scenario                                | DQL | Remark |
| --------------------------------------- | --- | ------ |
| Create submission                       | `INSERT INTO submissions (participant_id, test_question_id, answer, attempt_no, submitted_at) VALUES (1, 1, 'My answer', 1, NOW());` | `evaluated_result` and `score` filled later by evaluator |
| Get submission by id                    | `SELECT * FROM submissions WHERE id = 1;` |        |
| Get all submissions of participant      | `SELECT * FROM submissions WHERE participant_id = 1 ORDER BY submitted_at DESC;` |        |
| Get all submissions for test            | `SELECT s.* FROM submissions s JOIN test_questions tq ON tq.id = s.test_question_id WHERE tq.test_id = 1;` |        |
| Get all submissions for question        | `SELECT s.* FROM submissions s JOIN test_questions tq ON tq.id = s.test_question_id WHERE tq.question_id = 10;` |        |
| Get latest submission for question      | `SELECT * FROM submissions WHERE participant_id = 1 AND test_question_id = 1 ORDER BY attempt_no DESC LIMIT 1;` |        |
| Get highest scoring submission          | `SELECT * FROM submissions ORDER BY score DESC NULLS LAST LIMIT 1;` |        |
| Get submissions between dates           | `SELECT * FROM submissions WHERE submitted_at BETWEEN '2026-05-01' AND '2026-05-31';` |        |
| Count submissions per question          | `SELECT tq.question_id, COUNT(*) AS submission_count FROM submissions s JOIN test_questions tq ON tq.id = s.test_question_id GROUP BY tq.question_id;` |        |
| Count submissions per participant       | `SELECT participant_id, COUNT(*) AS submission_count FROM submissions GROUP BY participant_id;` |        |
| Get unanswered questions                | `SELECT tq.* FROM test_questions tq WHERE tq.test_id = 1 AND tq.id NOT IN (SELECT test_question_id FROM submissions WHERE participant_id = 1);` | For a given participant in a test |
| Get evaluated submissions               | `SELECT * FROM submissions WHERE evaluated_result IS NOT NULL;` |        |
| Get pending evaluations                 | `SELECT * FROM submissions WHERE evaluated_result IS NULL;` |        |
| Get submissions with score > X          | `SELECT * FROM submissions WHERE score > 5;` |        |
| Get submissions with failed test cases  | `SELECT * FROM submissions WHERE (evaluated_result->>'failed_count')::int > 0;` | Assumes evaluator stores `failed_count` in `evaluated_result` jsonb |
| Get submissions sorted by latest        | `SELECT * FROM submissions ORDER BY submitted_at DESC;` |        |
| Get average score per question          | `SELECT tq.question_id, AVG(s.score)::float AS avg_score FROM submissions s JOIN test_questions tq ON tq.id = s.test_question_id GROUP BY tq.question_id;` |        |
| Get average score per test              | `SELECT tq.test_id, AVG(s.score)::float AS avg_score FROM submissions s JOIN test_questions tq ON tq.id = s.test_question_id GROUP BY tq.test_id;` |        |
| Get submission history for participant  | `SELECT s.*, tq.question_id FROM submissions s JOIN test_questions tq ON tq.id = s.test_question_id WHERE s.participant_id = 1 ORDER BY s.submitted_at DESC;` |        |
| Get multiple attempts for same question | `SELECT * FROM submissions WHERE participant_id = 1 AND test_question_id = 1 ORDER BY attempt_no ASC;` |        |
| Get latest attempt only                 | `SELECT DISTINCT ON (participant_id, test_question_id) * FROM submissions ORDER BY participant_id, test_question_id, attempt_no DESC;` | One row per (participant, question) — the highest `attempt_no` |
| Get total score of participant          | `SELECT participant_id, SUM(score) AS total_score FROM (SELECT DISTINCT ON (participant_id, test_question_id) participant_id, test_question_id, score FROM submissions WHERE participant_id = 1 ORDER BY participant_id, test_question_id, attempt_no DESC) latest GROUP BY participant_id;` | Sums latest attempt per question only |
| Get participant rank in test            | `SELECT participant_id, total_score, RANK() OVER (ORDER BY total_score DESC) AS rank FROM (SELECT s.participant_id, SUM(s.score) AS total_score FROM submissions s JOIN test_questions tq ON tq.id = s.test_question_id WHERE tq.test_id = 1 GROUP BY s.participant_id) scores;` |        |

## 📊 Analytics / Reporting

| Scenario                    | DQL | Remark |
| --------------------------- | --- | ------ |
| Total users count           |     |        |
| Total tests count           |     |        |
| Total questions count       |     |        |
| Total submissions count     |     |        |
| Most active users           |     |        |
| Most attempted tests        |     |        |
| Most difficult questions    |     |        |
| Highest scoring users       |     |        |
| Lowest scoring questions    |     |        |
| Average score per test      |     |        |
| Average score per user      |     |        |
| Average completion time     |     |        |
| Test participation trends   |     |        |
| Daily registrations         |     |        |
| Daily submissions           |     |        |
| Monthly platform activity   |     |        |
| User growth statistics      |     |        |
| Top creators by tests       |     |        |
| Top creators by questions   |     |        |
| Submission trends over time |     |        |

## 🧠 Evaluation / AI Related

| Scenario                                | DQL | Remark |
| --------------------------------------- | --- | ------ |
| Get submissions pending evaluation      |     |        |
| Get submissions already evaluated       |     |        |
| Get evaluation result for submission    |     |        |
| Get failed test cases                   |     |        |
| Get partially correct submissions       |     |        |
| Re-evaluate submission                  |     |        |
| Get evaluation statistics               |     |        |
| Get average AI score confidence         |     |        |
| Get submissions needing manual review   |     |        |
| Compare multiple submission evaluations |     |        |
| Find inconsistent evaluations           |     |        |

## ⚡ Real-Time / Operational Scenarios

| Scenario                            | DQL | Remark |
| ----------------------------------- | --- | ------ |
| Get users currently taking test     |     |        |
| Auto-submit expired participants    |     |        |
| Find tests starting in next 10 mins |     |        |
| Find tests ending in next 10 mins   |     |        |
| Detect inactive participants        |     |        |
| Detect suspicious rapid submissions |     |        |
| Detect duplicate submissions        |     |        |
| Resume interrupted test session     |     |        |
| Get live leaderboard                |     |        |
| Get live submission feed            |     |        |

## 🧹 Integrity / Validation Queries

| Scenario                              | DQL | Remark |
| ------------------------------------- | --- | ------ |
| Find orphan submissions               |     |        |
| Find orphan mappings                  |     |        |
| Find users without roles              |     |        |
| Find tests without questions          |     |        |
| Find participants without submissions |     |        |
| Find duplicate usernames              |     |        |
| Find duplicate emails                 |     |        |
| Find duplicate question orders        |     |        |
| Validate policy mappings              |     |        |
| Find soft deleted records             |     |        |
| Find inconsistent timestamps          |     |        |

## 🔍 Complex Business Queries

| Scenario                                 | DQL | Remark |
| ---------------------------------------- | --- | ------ |
| Get tests a user can still attempt       |     |        |
| Get tests user failed                    |     |        |
| Get tests user passed                    |     |        |
| Get strongest topic of user              |     |        |
| Get weakest topic of user                |     |        |
| Recommend questions based on performance |     |        |
| Get users eligible for certificate       |     |        |
| Get top N users in test                  |     |        |
| Get percentile ranking                   |     |        |
| Compare performance between users        |     |        |
| Compare test difficulty                  |     |        |
| Detect cheating patterns                 |     |        |
| Detect unusually high scores             |     |        |
| Find abandoned tests                     |     |        |
| Get retry eligible participants          |     |        |
