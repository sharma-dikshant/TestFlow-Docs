---
title: API Design — Scenario Index
description: Maps business scenarios to planned API endpoints and entities; APIs to be defined in api-design/v2.md
status: draft
last_updated: 2026-05-05
---

# API Design — Scenario Index

> **Status: Work in Progress** — This document maps scenarios to APIs. Endpoint definitions live in [api-design/v2.md](./api-design/v2.md). The API and Entity Involved columns below are pending.

## Authentication and User Management

### 👤 Users & Auth

| Scenario                             | API | Entity Involved |
| ------------------------------------ | --- | --------------- |
| Get user by email                    | TBD | TBD |
| Get user by username                 | TBD | TBD |
| Login user                           | TBD | TBD |
| Check if email already exists        | TBD | TBD |
| Check if username already exists     | TBD | TBD |
| Get all active users                 | TBD | TBD |
| Get all verified users               | TBD | TBD |
| Get all users by role                | TBD | TBD |
| Get user profile with role           | TBD | TBD |
| Get recently registered users        | TBD | TBD |
| Soft delete user                     | TBD | TBD |
| Restore deleted user                 | TBD | TBD |
| Inactive User account                | TBD | TBD |
| Active User account                  | TBD | TBD |
| Search users by name/email           | TBD | TBD |
| Get all users created in last 7 days | TBD | TBD |

### 🔐 RBAC / Authorization

| Scenario                                      | API | Entity Involved |
| --------------------------------------------- | --- | --------------- |
| Get all roles                                 | TBD | TBD |
| Get all resources                             | TBD | TBD |
| Get all policies for a role                   | TBD | TBD |
| Get all resources accessible by a role        | TBD | TBD |
| Check whether role has access to resource     | TBD | TBD |
| Get all users having a specific role          | TBD | TBD |
| Get all roles using a specific resource       | TBD | TBD |
| Get all policies with role + resource details | TBD | TBD |
| Find unused resources                         | TBD | TBD |
| Find roles without policies                   | TBD | TBD |
| Get all permissions matrix for admin panel    | TBD | TBD |

## Test Management

### 📝 Tests

| Scenario                               | API | Entity Involved |
| -------------------------------------- | --- | --------------- |
| Create test                            | TBD | TBD |
| Get test by id                         | TBD | TBD |
| Get all tests                          | TBD | TBD |
| Get all upcoming tests                 | TBD | TBD |
| Get all ongoing tests                  | TBD | TBD |
| Get all completed tests                | TBD | TBD |
| Get all tests created by a user        | TBD | TBD |
| Search tests by title                  | TBD | TBD |
| Get test with creator details          | TBD | TBD |
| Get tests between dates                | TBD | TBD |
| Get tests with participant count       | TBD | TBD |
| Get tests with question count          | TBD | TBD |
| Get tests sorted by latest             | TBD | TBD |
| Get tests sorted by popularity         | TBD | TBD |
| Soft delete test                       | TBD | TBD |
| Get tests available for a user         | TBD | TBD |
| Get tests user already participated in | TBD | TBD |
| Get currently active tests             | TBD | TBD |
| Get tests ending soon                  | TBD | TBD |

### ❓ Questions

| Scenario                             | API | Entity Involved |
| ------------------------------------ | --- | --------------- |
| Create question                      | TBD | TBD |
| Get question by id                   | TBD | TBD |
| Get all questions                    | TBD | TBD |
| Get questions by type                | TBD | TBD |
| Get questions created by a user      | TBD | TBD |
| Search questions by text             | TBD | TBD |
| Get questions with default marks     | TBD | TBD |
| Get questions used in tests          | TBD | TBD |
| Get unused questions                 | TBD | TBD |
| Get most frequently used questions   | TBD | TBD |
| Get coding questions only            | TBD | TBD |
| Get MCQ questions only               | TBD | TBD |
| Get subjective questions only        | TBD | TBD |
| Get question with all tests using it | TBD | TBD |
| Get recently created questions       | TBD | TBD |

### 🔗 Test Questions Mapping

| Scenario                            | API | Entity Involved |
| ----------------------------------- | --- | --------------- |
| Add question to test                | TBD | TBD |
| Get all questions for a test        | TBD | TBD |
| Get ordered questions for a test    | TBD | TBD |
| Get total marks of a test           | TBD | TBD |
| Get question count for a test       | TBD | TBD |
| Get all tests containing a question | TBD | TBD |
| Update question order in test       | TBD | TBD |
| Update marks override in test       | TBD | TBD |
| Remove question from test           | TBD | TBD |
| Find duplicate question mappings    | TBD | TBD |
| Get first/last question of a test   | TBD | TBD |
| Get questions with custom marks     | TBD | TBD |

### 👥 Test Participants

| Scenario                               | API | Entity Involved |
| -------------------------------------- | --- | --------------- |
| Enroll user in test                    | TBD | TBD |
| Get all participants of a test         | TBD | TBD |
| Get all tests participated by a user   | TBD | TBD |
| Get participant status                 | TBD | TBD |
| Get participants currently taking test | TBD | TBD |
| Get completed participants             | TBD | TBD |
| Get absent participants                | TBD | TBD |
| Get participant start/completion time  | TBD | TBD |
| Get average completion time            | TBD | TBD |
| Count participants per test            | TBD | TBD |
| Count tests attempted by user          | TBD | TBD |
| Get leaderboard participation list     | TBD | TBD |
| Get latest participants                | TBD | TBD |
| Check if user already joined test      | TBD | TBD |
| Get participant by user + test         | TBD | TBD |
| Get participants who never submitted   | TBD | TBD |

### 📤 Submissions

| Scenario                                | API | Entity Involved |
| --------------------------------------- | --- | --------------- |
| Create submission                       | TBD | TBD |
| Get submission by id                    | TBD | TBD |
| Get all submissions of participant      | TBD | TBD |
| Get all submissions for test            | TBD | TBD |
| Get all submissions for question        | TBD | TBD |
| Get latest submission for question      | TBD | TBD |
| Get highest scoring submission          | TBD | TBD |
| Get submissions between dates           | TBD | TBD |
| Count submissions per question          | TBD | TBD |
| Count submissions per participant       | TBD | TBD |
| Get unanswered questions                | TBD | TBD |
| Get evaluated submissions               | TBD | TBD |
| Get pending evaluations                 | TBD | TBD |
| Get submissions with score > X          | TBD | TBD |
| Get submissions with failed test cases  | TBD | TBD |
| Get submissions sorted by latest        | TBD | TBD |
| Get average score per question          | TBD | TBD |
| Get average score per test              | TBD | TBD |
| Get submission history for participant  | TBD | TBD |
| Get multiple attempts for same question | TBD | TBD |
| Get latest attempt only                 | TBD | TBD |
| Get total score of participant          | TBD | TBD |
| Get participant rank in test            | TBD | TBD |
