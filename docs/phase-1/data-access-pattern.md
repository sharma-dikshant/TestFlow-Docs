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
| Create question                      |     |        |
| Get question by id                   |     |        |
| Get all questions                    |     |        |
| Get questions by type                |     |        |
| Get questions created by a user      |     |        |
| Search questions by text             |     |        |
| Get questions with default marks     |     |        |
| Get questions used in tests          |     |        |
| Get unused questions                 |     |        |
| Get most frequently used questions   |     |        |
| Get coding questions only            |     |        |
| Get MCQ questions only               |     |        |
| Get subjective questions only        |     |        |
| Get question with all tests using it |     |        |
| Get recently created questions       |     |        |

## 🔗 Test Questions Mapping

| Scenario                            | DQL | Remark |
| ----------------------------------- | --- | ------ |
| Add question to test                |     |        |
| Get all questions for a test        |     |        |
| Get ordered questions for a test    |     |        |
| Get total marks of a test           |     |        |
| Get question count for a test       |     |        |
| Get all tests containing a question |     |        |
| Update question order in test       |     |        |
| Update marks override in test       |     |        |
| Remove question from test           |     |        |
| Find duplicate question mappings    |     |        |
| Get first/last question of a test   |     |        |
| Get questions with custom marks     |     |        |

## 👥 Test Participants

| Scenario                               | DQL | Remark |
| -------------------------------------- | --- | ------ |
| Enroll user in test                    |     |        |
| Get all participants of a test         |     |        |
| Get all tests participated by a user   |     |        |
| Get participant status                 |     |        |
| Get participants currently taking test |     |        |
| Get completed participants             |     |        |
| Get absent participants                |     |        |
| Get participant start/completion time  |     |        |
| Get average completion time            |     |        |
| Count participants per test            |     |        |
| Count tests attempted by user          |     |        |
| Get leaderboard participation list     |     |        |
| Get latest participants                |     |        |
| Check if user already joined test      |     |        |
| Get participant by user + test         |     |        |
| Get participants who never submitted   |     |        |

## 📤 Submissions

| Scenario                                | DQL | Remark |
| --------------------------------------- | --- | ------ |
| Create submission                       |     |        |
| Get submission by id                    |     |        |
| Get all submissions of participant      |     |        |
| Get all submissions for test            |     |        |
| Get all submissions for question        |     |        |
| Get latest submission for question      |     |        |
| Get highest scoring submission          |     |        |
| Get submissions between dates           |     |        |
| Count submissions per question          |     |        |
| Count submissions per participant       |     |        |
| Get unanswered questions                |     |        |
| Get evaluated submissions               |     |        |
| Get pending evaluations                 |     |        |
| Get submissions with score > X          |     |        |
| Get submissions with failed test cases  |     |        |
| Get submissions sorted by latest        |     |        |
| Get average score per question          |     |        |
| Get average score per test              |     |        |
| Get submission history for participant  |     |        |
| Get multiple attempts for same question |     |        |
| Get latest attempt only                 |     |        |
| Get total score of participant          |     |        |
| Get participant rank in test            |     |        |

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
