# Non-functional requirements

Purpose: This document defines measurable quality requirements for ClinicQ and how students must verify them.

## NFR summary

| ID | Category | Requirement | Target | Verification method |
|---|---|---|---|---|
| NFR-PERF-01 | Performance | Availability latency in Must scope | p95 below 500 ms at 20 concurrent users with seed data | Local load test report |
| NFR-PERF-02 | Performance | Availability latency in Should scope | p95 below 300 ms at 50 concurrent users with seed data | Locust report |
| NFR-PERF-03 | Performance | Booking transaction latency | p95 below 800 ms for normal non-conflicting bookings | API performance test |
| NFR-SCALE-01 | Scalability | Seed data volume for local testing | 5 doctors, 20 patients, 2 weeks schedules, at least 100 appointments | Seed verification test |
| NFR-REL-01 | Reliability | Booking rules under race conditions | Same-slot test gives 1 success and 9 conflicts | Concurrency integration test |
| NFR-REL-02 | Reliability | Reminder delivery in Should scope | One reminder outbox row per appointment; delivery is at-least-once | Worker retry and redelivery tests |
| NFR-SEC-01 | Security | Password storage | Argon2 or bcrypt hashes only; no raw password persisted | Unit test and schema review |
| NFR-SEC-02 | Security | JWT token safety | Access 15 minutes; refresh 7 days; rotation and revocation enforced | Auth integration tests |
| NFR-SEC-03 | Security | Login throttling | 5 failures in 15 minutes causes 15-minute HTTP 429 lockout | Auth security tests |
| NFR-SEC-04 | Security | Authorization coverage | Every endpoint group x role pair is tested | Authorization test matrix |
| NFR-SEC-05 | Security | OWASP API risks | Broken auth, broken authorization, injection, and excessive data exposure are checked | Security checklist in PR |
| NFR-PRIV-01 | Privacy | Minimum personal data | Store name, e-mail, phone, role, appointment details only | Data model review |
| NFR-PRIV-02 | Privacy | DPDP Act 2023 awareness | Demo uses fictional data and documents purpose limitation | README privacy section |
| NFR-MAINT-01 | Maintainability | Type and lint quality | Ruff passes; mypy passes for application packages | CI checks |
| NFR-MAINT-02 | Maintainability | Test coverage | Total line at least 85%; total branch at least 75% | CI coverage gate |
| NFR-MAINT-03 | Maintainability | Service coverage | Booking, slots, and state machine at least 95% line and 85% branch | Targeted coverage report |
| NFR-OBS-01 | Observability | Request tracing in logs | Every request log has request ID, method, path, status, duration, and user ID when known | Log inspection test |
| NFR-OPS-01 | Portability | Local startup | Clean clone starts with documented commands on Windows WSL2, macOS, or Linux | Trainer setup check |
| NFR-OPS-02 | Hardware | Lite profile | 8 GB profile runs API, PostgreSQL, and tests with Celery, Redis, Mailpit, and Locust off | Trainer pre-check |
| NFR-LIC-01 | Licence compliance | Free local tools | Mandatory dependencies are open-source or free for local education use | Dependency review |
| NFR-API-01 | Usability | Client error clarity | Problem Details includes stable `type`, status, detail, and field errors | API error tests |

## Performance and capacity

ClinicQ is a learning project for one clinic. It does not need production-scale horizontal scaling in Must scope. It MUST still use indexes and query plans to avoid slow schedule pages.

| Requirement | Exact target | Evidence |
|---|---:|---|
| Availability endpoint p95 in Must scope | < 500 ms at 20 concurrent users | Performance run with seed data |
| Availability endpoint p95 in Should scope | < 300 ms at 50 concurrent users | Locust HTML or text report |
| Booking conflict response | < 1 second p95 under 10 same-slot parallel requests | Concurrency test timing |
| Search default page | `limit` default 20 and maximum 100 | API tests |
| Admin report | Completes in < 2 seconds with seed data and 1 month range | Report integration test |

The student MUST capture EXPLAIN ANALYZE evidence for the availability query and doctor performance report. The evidence MUST justify the indexes listed in `07-data-model.md`.

## Reliability

| Requirement | Target | Verification |
|---|---|---|
| Booking correctness | No duplicate active doctor slot can exist after committed transactions. | Database constraint test and concurrent API test |
| Patient appointment limit | At most 3 upcoming active appointments after concurrent requests. | 5 parallel booking test |
| Retry behavior | Serializable retry or lock retry handles safe transient conflicts. | Unit or integration test for selected ADR strategy |
| E-mail delivery | At-least-once only; no exactly-once promise. A duplicate e-mail is possible after SMTP succeeds and the worker crashes before marking sent. | Worker retry test and completed-task redelivery test |
| Health behavior | Liveness does not depend on DB; readiness checks DB. | Health endpoint tests |

## Security

ClinicQ MUST address the API parts of the OWASP Top 10 that matter for this project.

| Risk | ClinicQ control | Evidence |
|---|---|---|
| Broken authentication | Short access tokens, refresh rotation, token hashing, lockout | Auth tests |
| Broken object-level authorization | Own-patient and own-doctor checks on appointment reads and writes | Authorization matrix tests |
| Broken function-level authorization | Permission matrix for every endpoint group and role | API tests |
| Injection | Parameterized ORM or SQLAlchemy text with bound parameters | Code review checklist |
| Excessive data exposure | Patient responses exclude password hashes and other user records | API response tests |
| Security misconfiguration | `.env.example`, no debug stack traces in API errors | CI and manual check |
| Vulnerable dependencies | pip-audit and Trivy in CI | CI report |
| Secrets exposure | Gitleaks in pre-commit and CI | CI report |

Passwords MUST never be logged. Refresh token raw values MUST never be stored. Logs MUST not include Authorization headers.

## Privacy and data protection

ClinicQ stores minimum personal data for a clinic demo. The project MUST use fictional Indian names, fictional phone numbers, and `example.in` or `example.com` e-mail addresses. The student MUST mention India DPDP Act 2023 awareness in the implementation README. The README MUST explain purpose limitation, data minimization, and local demo data reset.

Audit logs can contain user IDs and appointment status changes. Audit logs MUST avoid medical details. Consultation notes are Should scope and MUST be limited to short visit notes and prescription text.

## Maintainability and testability

| Area | Requirement |
|---|---|
| Code style | Ruff lint and format MUST pass in CI. |
| Typing | mypy MUST pass for application packages. Tests and migrations MAY have relaxed rules. |
| Tests | pytest MUST run unit, API, integration, concurrency, security, and report tests. |
| Coverage | Total project coverage MUST be at least 85% line and 75% branch. |
| Service coverage | Booking rules, slot generation, and state machine MUST be at least 95% line and 85% branch. |
| ADRs | Students MUST document transaction strategy, token strategy, raw SQL report approach, e-mail worker choice, and deployment choice. |

## Observability

Each request log MUST be one JSON object. It MUST include `request_id`, `method`, `path`, `status_code`, `duration_ms`, `client_ip`, and `user_id` when authenticated. Appointment status-change logs MUST include appointment ID and new status. Logs MUST use UTC timestamps.

## Portability and hardware

Mandatory tooling MUST run on Windows 11 with WSL2 Ubuntu, macOS, or Linux. Docker image tags MUST be pinned. The project MUST not depend on paid cloud services for Must scope.

| Profile | Container set | Required limits |
|---|---|---|
| Lite 8 GB | API, PostgreSQL, test runner | Celery, Redis, Mailpit, Locust, and cloud demo are off. PostgreSQL target 512 MB. API target 512 MB. |
| Standard 16 GB | API, PostgreSQL, Redis, Celery, Mailpit, test runner | PostgreSQL target 1 GB. API target 1 GB. Worker target 512 MB. |

Windows WSL2 values: 8 GB laptops SHOULD use 4 GB memory and 4 GB swap. 16 GB laptops SHOULD use 8 GB memory and 4 GB swap. The trainer MUST run a lite profile pre-check before week 1.

## API usability

Errors MUST be predictable for future frontend developers. Problem Details `type` values used by ClinicQ include `email-already-registered`, `password-policy-failed`, `account-temporarily-locked`, `invalid-credentials`, `token-expired`, `invalid-token`, `account-deactivated`, `refresh-token-reused`, `refresh-token-revoked`, `permission-denied`, `doctor-not-active`, `specialization-not-active`, `fee-must-be-non-negative`, `too-many-sessions-for-weekday`, `sessions-overlap`, `booking-start-in-past`, `booking-window-exceeded`, `slot-not-on-schedule`, `slot-not-available`, `slot-already-booked`, `booking-limit-reached`, `patient-doctor-date-limit-reached`, `schedule-change-conflicts`, `invalid-status-transition`, `cancellation-window-closed`, `no-show-too-early`, `invalid-date-range`, `pagination-limit-exceeded`, `database-not-ready`, and `not-found`.

## Licence compliance

Mandatory tools MUST be free for local student use. The student MUST list direct dependencies in `pyproject.toml` and commit `uv.lock`. The student MUST avoid proprietary sample data and real clinic data.

[Back to README](../README.md)
