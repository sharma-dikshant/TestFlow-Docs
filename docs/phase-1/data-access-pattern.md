# Data Access Patterns

## 👤 Users & Auth

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

| Scenario                                      | DQL | Remark |
| --------------------------------------------- | --- | ------ |
| Get all roles                                 |     |        |
| Get all resources                             |     |        |
| Get all policies for a role                   |     |        |
| Get all resources accessible by a role        |     |        |
| Check whether role has access to resource     |     |        |
| Get all users having a specific role          |     |        |
| Get all roles using a specific resource       |     |        |
| Get all policies with role + resource details |     |        |
| Find unused resources                         |     |        |
| Find roles without policies                   |     |        |
| Get all permissions matrix for admin panel    |     |        |

## 📝 Tests

| Scenario                               | DQL | Remark |
| -------------------------------------- | --- | ------ |
| Create test                            |     |        |
| Get test by id                         |     |        |
| Get all tests                          |     |        |
| Get all upcoming tests                 |     |        |
| Get all ongoing tests                  |     |        |
| Get all completed tests                |     |        |
| Get all tests created by a user        |     |        |
| Search tests by title                  |     |        |
| Get test with creator details          |     |        |
| Get tests between dates                |     |        |
| Get tests with participant count       |     |        |
| Get tests with question count          |     |        |
| Get tests sorted by latest             |     |        |
| Get tests sorted by popularity         |     |        |
| Soft delete test                       |     |        |
| Get tests available for a user         |     |        |
| Get tests user already participated in |     |        |
| Get currently active tests             |     |        |
| Get tests ending soon                  |     |        |

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
