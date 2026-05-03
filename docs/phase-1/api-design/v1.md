# API Design

## Authentication and User Management

### 👤 Users & Auth

| Scenario                             | API | Entity Involved |
| ------------------------------------ | --- | --------------- |
| Get user by email                    |     |                 |
| Get user by username                 |     |                 |
| Login user                           |     |                 |
| Check if email already exists        |     |                 |
| Check if username already exists     |     |                 |
| Get all active users                 |     |                 |
| Get all verified users               |     |                 |
| Get all users by role                |     |                 |
| Get user profile with role           |     |                 |
| Get recently registered users        |     |                 |
| Soft delete user                     |     |                 |
| Restore deleted user                 |     |                 |
| Inactive User account                |     |                 |
| Active User account                  |     |                 |
| Search users by name/email           |     |                 |
| Get all users created in last 7 days |     |                 |

### 🔐 RBAC / Authorization

| Scenario                                      | API | Entity Involved |
| --------------------------------------------- | --- | --------------- |
| Get all roles                                 |     |                 |
| Get all resources                             |     |                 |
| Get all policies for a role                   |     |                 |
| Get all resources accessible by a role        |     |                 |
| Check whether role has access to resource     |     |                 |
| Get all users having a specific role          |     |                 |
| Get all roles using a specific resource       |     |                 |
| Get all policies with role + resource details |     |                 |
| Find unused resources                         |     |                 |
| Find roles without policies                   |     |                 |
| Get all permissions matrix for admin panel    |     |                 |

## Test Management

### 📝 Tests

| Scenario                               | API | Entity Involved |
| -------------------------------------- | --- | --------------- |
| Create test                            |     |                 |
| Get test by id                         |     |                 |
| Get all tests                          |     |                 |
| Get all upcoming tests                 |     |                 |
| Get all ongoing tests                  |     |                 |
| Get all completed tests                |     |                 |
| Get all tests created by a user        |     |                 |
| Search tests by title                  |     |                 |
| Get test with creator details          |     |                 |
| Get tests between dates                |     |                 |
| Get tests with participant count       |     |                 |
| Get tests with question count          |     |                 |
| Get tests sorted by latest             |     |                 |
| Get tests sorted by popularity         |     |                 |
| Soft delete test                       |     |                 |
| Get tests available for a user         |     |                 |
| Get tests user already participated in |     |                 |
| Get currently active tests             |     |                 |
| Get tests ending soon                  |     |                 |

### ❓ Questions

| Scenario                             | API | Entity Involved |
| ------------------------------------ | --- | --------------- |
| Create question                      |     |                 |
| Get question by id                   |     |                 |
| Get all questions                    |     |                 |
| Get questions by type                |     |                 |
| Get questions created by a user      |     |                 |
| Search questions by text             |     |                 |
| Get questions with default marks     |     |                 |
| Get questions used in tests          |     |                 |
| Get unused questions                 |     |                 |
| Get most frequently used questions   |     |                 |
| Get coding questions only            |     |                 |
| Get MCQ questions only               |     |                 |
| Get subjective questions only        |     |                 |
| Get question with all tests using it |     |                 |
| Get recently created questions       |     |                 |

### 🔗 Test Questions Mapping

| Scenario                            | API | Entity Involved |
| ----------------------------------- | --- | --------------- |
| Add question to test                |     |                 |
| Get all questions for a test        |     |                 |
| Get ordered questions for a test    |     |                 |
| Get total marks of a test           |     |                 |
| Get question count for a test       |     |                 |
| Get all tests containing a question |     |                 |
| Update question order in test       |     |                 |
| Update marks override in test       |     |                 |
| Remove question from test           |     |                 |
| Find duplicate question mappings    |     |                 |
| Get first/last question of a test   |     |                 |
| Get questions with custom marks     |     |                 |

### 👥 Test Participants

| Scenario                               | API | Entity Involved |
| -------------------------------------- | --- | --------------- |
| Enroll user in test                    |     |                 |
| Get all participants of a test         |     |                 |
| Get all tests participated by a user   |     |                 |
| Get participant status                 |     |                 |
| Get participants currently taking test |     |                 |
| Get completed participants             |     |                 |
| Get absent participants                |     |                 |
| Get participant start/completion time  |     |                 |
| Get average completion time            |     |                 |
| Count participants per test            |     |                 |
| Count tests attempted by user          |     |                 |
| Get leaderboard participation list     |     |                 |
| Get latest participants                |     |                 |
| Check if user already joined test      |     |                 |
| Get participant by user + test         |     |                 |
| Get participants who never submitted   |     |                 |

### 📤 Submissions

| Scenario                                | API | Entity Involved |
| --------------------------------------- | --- | --------------- |
| Create submission                       |     |                 |
| Get submission by id                    |     |                 |
| Get all submissions of participant      |     |                 |
| Get all submissions for test            |     |                 |
| Get all submissions for question        |     |                 |
| Get latest submission for question      |     |                 |
| Get highest scoring submission          |     |                 |
| Get submissions between dates           |     |                 |
| Count submissions per question          |     |                 |
| Count submissions per participant       |     |                 |
| Get unanswered questions                |     |                 |
| Get evaluated submissions               |     |                 |
| Get pending evaluations                 |     |                 |
| Get submissions with score > X          |     |                 |
| Get submissions with failed test cases  |     |                 |
| Get submissions sorted by latest        |     |                 |
| Get average score per question          |     |                 |
| Get average score per test              |     |                 |
| Get submission history for participant  |     |                 |
| Get multiple attempts for same question |     |                 |
| Get latest attempt only                 |     |                 |
| Get total score of participant          |     |                 |
| Get participant rank in test            |     |                 |
