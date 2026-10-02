# Interview preparation

Purpose: This document helps a student explain ClinicQ in interviews and connect project work to backend job skills.

## Explain your project in 2 minutes using STAR

| STAR part | ClinicQ talking points |
|---|---|
| Situation | A Pune clinic used phone and paper booking. Staff had double bookings, long queues, and poor no-show visibility. |
| Task | I built a FastAPI REST API for appointment booking with JWT auth, PostgreSQL rules, and role-based workflows. |
| Action | I implemented doctor schedules, availability, concurrency-safe booking, appointment states, audit logs, and a SQL report. I used tests, Docker Compose, CI, and a Bruno collection. |
| Result | The system prevents duplicate active slots, limits each patient to 3 upcoming appointments, reports doctor revenue, and starts from a clean clone. |

## Interview questions by topic

### REST and HTTP

1. What makes ClinicQ a REST API?
   - Resource paths use `/api/v1`.
   - HTTP methods match actions.
   - Status codes carry meaning.
2. Why return 409 for a booked slot?
   - The request is syntactically valid.
   - Current resource state conflicts.
   - `slot-already-booked` explains the conflict.
3. When does ClinicQ use 422?
   - Password policy failure.
   - `limit=101` pagination error.
   - Invalid date or slot alignment.
4. Why use RFC 9457 Problem Details?
   - One error shape helps clients.
   - Stable `type` values support tests.
   - Field errors can be nested.
5. Why version the API as `/api/v1`?
   - Future clients can migrate safely.
   - Breaking changes get a new prefix.
   - OpenAPI stays clear.

### Authentication and security

6. Why are access tokens short-lived?
   - A stolen token expires quickly.
   - ClinicQ uses 15 minutes.
   - Refresh token handles session continuity.
7. How does refresh token rotation work?
   - Each refresh returns a new refresh token.
   - Old token is marked rotated.
   - Reuse revokes the token family.
8. Why hash passwords with Argon2 or bcrypt?
   - Raw passwords must never be stored.
   - Slow hashes reduce brute-force value.
   - Salted hashes handle duplicate passwords.
9. How does login throttling work?
   - Count 5 failures in 15 minutes.
   - Lock for 15 minutes.
   - Return HTTP 429.
10. What is RBAC in ClinicQ?
   - Roles are Patient, Doctor, Receptionist, Admin.
   - Endpoint groups check roles.
   - Own-resource checks are still needed.

### SQL and data modeling

11. Why store timestamps in UTC?
   - UTC avoids daylight and server-location surprises.
   - IST display is derived.
   - Reports can use consistent ranges.
12. Why keep `local_date` on appointments?
   - Patient-doctor-date rule uses Asia/Kolkata.
   - UTC date can differ near midnight.
   - Indexing local date helps checks.
13. What is a partial unique index?
   - It applies only to rows matching a condition.
   - Active statuses block doctor slots.
   - Cancelled slots can be reused.
14. What is a window function?
   - It computes across related result rows.
   - ClinicQ ranks doctors by revenue.
   - Rows remain visible after ranking.
15. What does EXPLAIN ANALYZE show?
   - Actual query plan.
   - Time spent per step.
   - Whether indexes are used.
16. Why use a transaction for booking?
   - Multiple checks must commit together.
   - Race conditions need database protection.
   - Audit and appointment rows must stay consistent.

### Concurrency and reliability

17. What race condition can happen in booking?
   - Two requests see the same free slot.
   - Both try to insert active appointments.
   - Only one must commit successfully.
18. How can PostgreSQL help concurrency?
   - Unique constraints reject duplicates.
   - Row locks serialize critical reads.
   - SERIALIZABLE isolation can retry conflicts.
19. Why is in-memory locking not enough?
   - Multiple API workers do not share memory.
   - A restart loses locks.
   - The database is the shared authority.
20. What does idempotency mean for reminders?
   - Same reminder key can be retried.
   - Sent outbox rows are skipped.
   - Ambiguous SMTP success can still cause duplicate e-mail.

### FastAPI and Python

21. Why use Pydantic models?
   - Validate request bodies.
   - Document response shapes.
   - Prevent accidental extra fields.
22. What belongs in a service layer?
   - Booking rules.
   - State transitions.
   - Transaction decisions.
23. What is dependency injection in FastAPI?
   - Functions provide DB sessions or current users.
   - Tests can override dependencies.
   - Handlers stay small.
24. Why run mypy on application packages?
   - Detect wrong optional handling.
   - Catch DTO and service mismatches.
   - Improve maintainability.

### Testing and DevOps

25. What test proves double-booking safety?
   - 10 parallel requests target one slot.
   - Exactly 1 returns 201.
   - Exactly 9 return 409.
26. Why use Testcontainers for PostgreSQL tests?
   - Tests use real PostgreSQL behavior.
   - Partial indexes and locks are exercised.
   - SQLite differences are avoided.
27. What should the CI pipeline run?
   - Ruff, mypy, pytest, coverage.
   - Docker build.
   - pip-audit, Trivy, Gitleaks.
28. Why use Docker Compose?
   - One command starts local services.
   - API, PostgreSQL, Mailpit, Redis fit demos.
   - Profiles support 8 GB laptops.
29. What should not be logged?
   - Passwords.
   - Raw refresh tokens.
   - Authorization headers.
30. Why commit `uv.lock`?
   - Reproducible dependencies.
   - Faster clean setup.
   - CI and laptop use the same versions.

## Project deep-dive questions

1. How did you enforce maximum 3 upcoming active appointments?
   - Count active future appointments in the same transaction.
   - Protect concurrent requests with selected strategy.
   - Return `booking-limit-reached` on conflict.
2. How did you reject schedule changes that hit active appointments?
   - Compare new leave or session with active appointment times.
   - Return 409 with affected IDs.
   - Do not modify the schedule.
3. How did you design the audit log?
   - Store appointment ID, actor, from, to, reason, timestamp.
   - Write audit row in the same transaction.
   - Do not expose update or delete endpoints.
4. How did you separate patient and receptionist booking?
   - Patient can book only self.
   - Receptionist can book registered patients.
   - Both paths use the same service rules.
5. How did you prove query performance?
   - Seed expected data.
   - Run EXPLAIN ANALYZE on availability and report queries.
   - Add indexes justified by plans.
6. How does cancellation reopen a slot?
   - `CANCELLED` is not active.
   - Active-only uniqueness ignores cancelled rows.
   - Cancel-then-book test returns 201.
7. Why is readiness different from liveness?
   - Liveness checks API process.
   - Readiness checks database access.
   - Orchestrators can route only to ready API.
8. What trade-off did your transaction ADR choose?
   - State chosen strategy.
   - Explain retry or lock behavior.
   - Describe test evidence.

## Fundamentals check

### Python core

- Explain mutable and immutable objects using appointment payload examples.
- Explain exceptions and why service errors map to HTTP errors.
- Explain context managers for database sessions.
- Explain type hints for request DTOs and service returns.

### SQL

- Explain INNER JOIN between doctors and appointments.
- Explain GROUP BY for completed count and revenue.
- Explain window functions for revenue rank.
- Explain transactions, isolation levels, locks, and indexes.
- Explain why bound parameters prevent SQL injection.

### Git

- Explain branch, commit, merge, and pull request.
- Explain Conventional Commits.
- Explain why direct push to `main` is blocked.
- Explain how to revert a bad change.

### Linux

- Explain `pwd`, `ls`, `cd`, `grep`, `curl`, and environment variables.
- Explain file permissions at a basic level.
- Explain how to inspect container logs.
- Explain why WSL2 files should stay inside the Linux filesystem.

### HTTP and networking

- Explain DNS, TCP, port, host, and URL.
- Explain 200, 201, 401, 403, 409, 422, 429, and 503.
- Explain request headers and JSON body.
- Explain CORS for a future frontend.

### Local tools mapped to AWS services

| Local tool | AWS concept to mention |
|---|---|
| Docker Compose API container | ECS task or Elastic Beanstalk app |
| PostgreSQL | Amazon RDS for PostgreSQL |
| Redis | Amazon ElastiCache for Redis |
| Mailpit | Amazon SES sandbox concept |
| GitHub Actions | CodeBuild or CodePipeline concept |
| Trivy and pip-audit | ECR scanning and dependency scanning concept |

## Resume bullet templates

- Built `ClinicQ`, a FastAPI appointment API with JWT auth, refresh rotation, RBAC, and PostgreSQL-backed booking rules; achieved `<line>%` line and `<branch>%` branch coverage.
- Prevented double booking with `<strategy>` and tests where 10 parallel same-slot requests produced 1 success and 9 conflicts.
- Designed PostgreSQL schema with active-only uniqueness, audit logs, and EXPLAIN ANALYZE-backed indexes for availability and revenue reports.
- Containerized API, PostgreSQL, Mailpit, Redis, and Celery with Docker Compose profiles for 8 GB and 16 GB laptops.
- Created a Bruno API collection and CI pipeline with Ruff, mypy, pytest, coverage, pip-audit, Trivy, and Gitleaks.

## GitHub and LinkedIn tips

- Pin the `clinicq-api` repository on GitHub.
- Keep README setup steps short and tested.
- Link the final demo video and screenshots of CI, coverage, and Bruno runs.
- Write a LinkedIn post about one hard problem: double booking or refresh token reuse.
- Do not claim production healthcare compliance. Say it is a local learning project.

## Mock-interview checklist

- Explain the project in 2 minutes without reading.
- Draw the booking transaction on paper.
- Explain one failed test and how it was fixed.
- Show one SQL query plan and one index decision.
- Answer the difference between 401, 403, 409, and 422.
- Explain every line of one appointment service function.
- Run the demo collection from a clean local stack.
- State one improvement you would make with more time.

[Back to README](../README.md)
