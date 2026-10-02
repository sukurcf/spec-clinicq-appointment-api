# Project overview

Purpose: This document explains the ClinicQ problem, project scope, success measures, constraints, and job relevance.

## Problem statement

A fictional multi-doctor clinic in Pune books appointments by phone and on paper. Staff can book the same doctor and slot twice. Patients wait in long queues because arrival status is not visible. Doctors do not have a reliable day schedule. The clinic admin cannot measure completed appointments, no-shows, or revenue by doctor.

ClinicQ is a Python backend-only REST API operated through authenticated API requests. Python tests or a local API client prove booking rules, clear HTTP contracts, and database-backed concurrency control. No business interface is implemented.

## Vision

ClinicQ will become the single appointment source for the clinic. It will show doctors and staff the same schedule. It will reject conflicting bookings with RFC 9457 Problem Details. It will record every appointment status change in an audit log.

## Measurable goals

| Goal | Target | Evidence |
|---|---:|---|
| Prevent double booking | 10 parallel requests for one doctor slot produce exactly 1 HTTP 201 and 9 HTTP 409 responses | Concurrency integration test |
| Enforce patient limit | A patient has at most 3 upcoming active appointments | Service tests and API tests |
| Keep one patient-doctor-date booking | A patient has at most 1 active appointment with the same doctor on one IST date | Booking rule tests |
| Make availability responsive | MUST availability endpoint p95 is below 500 ms at 20 concurrent users with seed data | Local performance run |
| Meet Should load goal | SHOULD availability endpoint p95 is below 300 ms at 50 concurrent users with seed data | Locust report |
| Test business logic deeply | Booking rules, slot generation, and state machine have at least 95% line coverage | Coverage report |
| Meet project coverage | Total line coverage is at least 85%; branch coverage is at least 75% | CI coverage gate |
| Keep setup short | A clean clone starts in 10 documented steps or fewer through one entry point | TC-LOCAL-001 and trainer setup check |
| Prove local independence | After initial downloads, recorded fixtures and core operations need no external network; stop/start preserves appointments and audit rows | TC-LOCAL-002 and TC-LOCAL-003 |

## Scope by priority

### Must scope

- Patient self-registration, login, JWT access tokens, refresh token rotation, logout, and lockout.
- Admin-created staff accounts and role-based access control.
- Doctor, specialization, fee, weekly session, leave, and holiday management.
- Availability calculation for fixed 15-minute slots.
- Patient and receptionist booking with all concurrency-safe booking rules.
- Appointment state changes: `BOOKED`, `CHECKED_IN`, `COMPLETED`, `CANCELLED`, and `NO_SHOW`.
- Patient, receptionist, doctor, and admin schedule REST responses required by the brief.
- Doctor search by specialization, partial name, and available date.
- `/api/v1` REST API with OpenAPI and RFC 9457 Problem Details.
- Health endpoints, migrations, fictional seed/recorded fixtures, audit log, JSON logs, request IDs, and persisted local lifecycle.
- One hand-written SQL doctor performance report with query-plan evidence.
- Bruno/Postman collection or Python HTTP demonstration for the demo.

### Should scope

- Booking and cancellation e-mail through local Mailpit.
- Reminder e-mail 24 hours before appointment time through local Celery and Redis.
- Per-doctor slot length of 10, 15, 20, or 30 minutes.
- Walk-in patient records without login accounts.
- Consultation notes and prescription text.
- Extra admin reports, CSV export, expanded API error-contract regressions, and local Locust load testing.

### Could scope

- Atomic reschedule.
- Waitlist notification for a freed slot.
- Async SQLAlchemy or Redis availability cache with safe invalidation.
- Schemathesis contract testing.

## Non-goals and out of scope

Payments, video consultation, full medical records, SMS, WhatsApp, student-built frontend of any kind, multi-clinic tenancy, insurance, public deployment, and real patient data are out of scope, even as bonus work. ClinicQ does not diagnose patients. ClinicQ does not store lab reports or medical images. Built-in local API documentation and Mailpit consoles may be inspected, but no student implements or styles them.

## Assumptions

- The clinic uses one time zone for display: Asia/Kolkata.
- All persisted timestamps use UTC.
- Active appointment statuses are exactly `BOOKED` and `CHECKED_IN`.
- A cancelled slot can be booked again.
- The MUST slot length is fixed at 15 minutes.
- The student uses PostgreSQL 18.x and records the concurrency choice in an ADR.
- The seed command creates 5 doctors, 20 patients, and 2 weeks of schedules.

## Constraints

| Constraint | Requirement |
|---|---|
| Time budget | MUST scope MUST be about 120 hours or less, including about 30 hours of guided learning and setup. |
| Total project effort | The full 6-week project is about 180 hours. |
| Hardware | MUST scope runs on an 8 GB, four-core laptop with 20 GB free disk through the lite profile; 16 GB is recommended. |
| Local-first tools | Mandatory services MUST run locally and be free. |
| API prefix | All product API endpoints MUST start with `/api/v1`. |
| Privacy | Use fictional Indian names and `example.in` or `example.com` addresses. |
| Docker tags | Image tags MUST be pinned to versions. Never use `latest`. |

## Effort budget statement

The ClinicQ MUST scope is designed for about 120 hours. This includes about 30 hours for guided learning, environment setup, API basics, JWT concepts, PostgreSQL transactions, and test setup. The remaining about 60 hours of the 6-week schedule are for Should items, hardening, interview preparation, documentation, and contingency. Could items start only after all Must requirements pass.

## Hardware profiles

| Profile | Use | Required limits |
|---|---|---|
| Lite 8 GB | Mandatory minimum | Run API, PostgreSQL, and test runner. Switch off local Celery, Redis, Mailpit, and Locust unless testing a related Should item. PostgreSQL memory target is 512 MB. API container target is 512 MB. |
| Standard 16 GB | Recommended | Run API, PostgreSQL, Redis, Celery, Mailpit, and test runner together. PostgreSQL memory target is 1 GB. API container target is 1 GB. |

Windows users SHOULD configure WSL2 with 4 GB memory and 4 GB swap on 8 GB laptops. They SHOULD use 8 GB memory and 4 GB swap on 16 GB laptops. The trainer MUST validate the lite profile on an 8 GB laptop before week 1.

Resource figures are proposed limits, not measured application results. Document 06 supplies the single local start/stop, fixture, port, reset, and pre-check contract.

## Success criteria

1. All Must FRs are implemented and demonstrated through the local API collection or Python HTTP flows.
2. CI on `main` is green.
3. Coverage gates are met.
4. Business-rule negative tests include every booking rule.
5. Authorization tests cover every endpoint group and role.
6. Query-plan notes justify indexes for availability and doctor performance report.
7. No secrets or real personal data appear in the repository history.
8. The student can explain JWT rotation, lockout, isolation, indexes, and HTTP status codes.
9. All local acceptance cases in document 09 pass, including offline fixture use and persisted appointment/audit/token state.

## Skills learned and job relevance

| Skill | ClinicQ practice | Job-description keywords |
|---|---|---|
| REST API design | Versioned appointment endpoints and Problem Details | FastAPI, REST, OpenAPI, HTTP status codes |
| Authentication | JWT access token, refresh rotation, lockout | JWT, password hashing, RBAC |
| Database design | Appointments, partial uniqueness, audit logs | PostgreSQL, indexes, constraints, Alembic |
| Concurrency | Double-booking protection under parallel requests | transactions, isolation levels, race conditions |
| Testing | Unit, API, integration, concurrency, coverage | pytest, Testcontainers, coverage |
| SQL depth | Doctor performance report with ranking | joins, aggregation, window functions, EXPLAIN ANALYZE |
| DevOps basics | Docker Compose and CI quality gates | Docker, GitHub Actions, uv, pre-commit |
| Observability | JSON logs with request IDs | structured logging, request tracing |

[Back to README](../README.md)
