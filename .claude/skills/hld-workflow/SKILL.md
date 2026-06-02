# LLD Workflow

A step-by-step process for designing any feature at the low-level.

---

## 1. Pick Feature

Choose **one feature only**.

> Examples: Join Test, Create Test, Submit Answer

---

## 2. Define Business Goal

- Why does this feature exist?
- What problem does it solve?

---

## 3. Define Business Rules / Invariants

Rules that must always remain true.

> Examples:
>
> - User cannot join twice
> - Test cannot exceed max participants
> - User cannot submit after test ends

---

## 4. Define State Transitions

- Identify valid state flow
- Define invalid transitions as well

> Example: `NOT_STARTED → IN_PROGRESS → COMPLETED`

---

## 5. Define Feature Flow

- Happy path
- Edge cases
- Failure scenarios

---

## 6. Define API

| Field        | Details        |
| ------------ | -------------- |
| Method       | GET / POST / … |
| Endpoint     | `/resource`    |
| Query params | `?key=value`   |
| Route params | `/:id`         |

---

## 7. Define Request / Response

- Request body
- Response body
- Status codes
- Error response structure

---

## 8. Define Validation Layer

- API-level validation
- DTO / schema validation
- Input sanitization

---

## 9. Define Authorization

- Which role / resource can access?
- Ownership checks
- Policy checks

---

## 10. Break Feature into Layers

### Controller

- Handle HTTP request / response
- Call service
- Return formatted response

### Service

- Contains business logic
- Orchestrates feature flow
- Module-to-module communication only through service

### Repository

- Database queries only
- No business logic

---

## 11. Define Service Logic

- Main orchestration logic
- Conditions and decision-making
- Workflow execution

---

## 12. Define Transactions

- Identify atomic operations
- Define rollback strategy

> Example atomic pair: reserve participant slot → create participant

---

## 13. Think About Concurrency

- Parallel requests
- Duplicate requests
- Race conditions
- Locking / atomic update strategy

---

## 14. Define Error Handling

- Business errors
- Validation errors
- DB errors
- Standard error codes / messages

---

## 15. Define Side Effects

- Notifications
- Emails
- Events
- Analytics
- Logs

---

## 16. Define Observability

- Logging
- Metrics
- Audit trails
- Monitoring points

---

## 17. Define Performance Considerations

- Query optimization
- Indexing
- Caching
- N+1 query prevention

---

## 18. Define Extensibility

- Future-proofing
- Can new states / features be added easily?
- Can the module scale independently?

---

## 19. Define Testing Strategy

- Unit tests
- Integration tests
- Edge case tests
- Failure tests
