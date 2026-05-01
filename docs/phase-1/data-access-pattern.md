# Data Access Patterns

## 👤 Users & Auth

- Get user by email
- Get user by username
- Login user
- Check if email already exists
- Check if username already exists
- Get all active users
- Get all verified users
- Get all users by role
- Get user profile with role
- Get recently registered users
- Soft delete user
- Restore deleted user
- Count users by status
- Search users by name/email
- Get all users created in last 7 days

## 🔐 RBAC / Authorization

- Get all roles
- Get all resources
- Get all policies for a role
- Get all resources accessible by a role
- Check whether role has access to resource
- Get all users having a specific role
- Get all roles using a specific resource
- Get all policies with role + resource details
- Find unused resources
- Find roles without policies
- Get all permissions matrix for admin panel

## 📝 Tests

- Create test
- Get test by id
- Get all tests
- Get all upcoming tests
- Get all ongoing tests
- Get all completed tests
- Get all tests created by a user
- Search tests by title
- Get test with creator details
- Get tests between dates
- Get tests with participant count
- Get tests with question count
- Get tests sorted by latest
- Get tests sorted by popularity
- Soft delete test
- Get tests available for a user
- Get tests user already participated in
- Get currently active tests
- Get tests ending soon

## ❓ Questions

- Create question
- Get question by id
- Get all questions
- Get questions by type
- Get questions created by a user
- Search questions by text
- Get questions with default marks
- Get questions used in tests
- Get unused questions
- Get most frequently used questions
- Get coding questions only
- Get MCQ questions only
- Get subjective questions only
- Get question with all tests using it
- Get recently created questions

## 🔗 Test Questions Mapping

- Add question to test
- Get all questions for a test
- Get ordered questions for a test
- Get total marks of a test
- Get question count for a test
- Get all tests containing a question
- Update question order in test
- Update marks override in test
- Remove question from test
- Find duplicate question mappings
- Get first/last question of a test
- Get questions with custom marks

## 👥 Test Participants

- Enroll user in test
- Get all participants of a test
- Get all tests participated by a user
- Get participant status
- Get participants currently taking test
- Get completed participants
- Get absent participants
- Get participant start/completion time
- Get average completion time
- Count participants per test
- Count tests attempted by user
- Get leaderboard participation list
- Get latest participants
- Check if user already joined test
- Get participant by user + test
- Get participants who never submitted

## 📤 Submissions

- Create submission
- Get submission by id
- Get all submissions of participant
- Get all submissions for test
- Get all submissions for question
- Get latest submission for question
- Get highest scoring submission
- Get submissions between dates
- Count submissions per question
- Count submissions per participant
- Get unanswered questions
- Get evaluated submissions
- Get pending evaluations
- Get submissions with score > X
- Get submissions with failed test cases
- Get submissions sorted by latest
- Get average score per question
- Get average score per test
- Get submission history for participant
- Get multiple attempts for same question
- Get latest attempt only
- Get total score of participant
- Get participant rank in test

## 📊 Analytics / Reporting

- Total users count
- Total tests count
- Total questions count
- Total submissions count
- Most active users
- Most attempted tests
- Most difficult questions
- Highest scoring users
- Lowest scoring questions
- Average score per test
- Average score per user
- Average completion time
- Test participation trends
- Daily registrations
- Daily submissions
- Monthly platform activity
- User growth statistics
- Top creators by tests
- Top creators by questions
- Submission trends over time

## 🧠 Evaluation / AI Related

- Get submissions pending evaluation
- Get submissions already evaluated
- Get evaluation result for submission
- Get failed test cases
- Get partially correct submissions
- Re-evaluate submission
- Get evaluation statistics
- Get average AI score confidence
- Get submissions needing manual review
- Compare multiple submission evaluations
- Find inconsistent evaluations

## ⚡ Real-Time / Operational Scenarios

- Get users currently taking test
- Auto-submit expired participants
- Find tests starting in next 10 mins
- Find tests ending in next 10 mins
- Detect inactive participants
- Detect suspicious rapid submissions
- Detect duplicate submissions
- Resume interrupted test session
- Get live leaderboard
- Get live submission feed

## 🧹 Integrity / Validation Queries

- Find orphan submissions
- Find orphan mappings
- Find users without roles
- Find tests without questions
- Find participants without submissions
- Find duplicate usernames
- Find duplicate emails
- Find duplicate question orders
- Validate policy mappings
- Find soft deleted records
- Find inconsistent timestamps

## 🔍 Complex Business Queries

- Get tests a user can still attempt
- Get tests user failed
- Get tests user passed
- Get strongest topic of user
- Get weakest topic of user
- Recommend questions based on performance
- Get users eligible for certificate
- Get top N users in test
- Get percentile ranking
- Compare performance between users
- Compare test difficulty
- Detect cheating patterns
- Detect unusually high scores
- Find abandoned tests
- Get retry eligible participants
