# System architecture

Purpose: This document defines the expected architecture for the `clinicq-api` student implementation.

## Context diagram

```mermaid
flowchart LR
  Patient[Patient] --> API[ClinicQ REST API]
  Doctor[Doctor] --> API
  Receptionist[Receptionist] --> API
  Admin[Clinic Admin] --> API
  Bruno[Bruno or Postman collection] --> API
  API --> DB[(PostgreSQL)]
  API --> Mailpit[Mailpit SMTP and inbox]
  API --> Redis[(Redis)]
  Worker[Celery notification worker] --> Redis
  Worker --> DB
  Worker --> Mailpit
  API --> Logs[JSON logs with request ID]
```

ClinicQ is a backend-only appointment booking API. Patients, doctors, receptionists, and admins use future clients or the demo API collection. The API owns authentication, role checks, booking rules, availability generation, and reports. PostgreSQL is the source of truth for users, doctors, schedules, leaves, appointments, audit logs, refresh tokens, login attempts, and notification outbox rows.

## Container diagram

```mermaid
flowchart TB
  Client[API client] -->|HTTPS or local HTTP| FastAPI[FastAPI app]
  FastAPI --> Auth[Auth and RBAC services]
  FastAPI --> Appointment[Appointment service]
  FastAPI --> Availability[Availability service]
  FastAPI --> Report[Report service]
  Auth --> Postgres[(PostgreSQL 18.x)]
  Appointment --> Postgres
  Availability --> Postgres
  Report --> Postgres
  Appointment --> Outbox[Notification outbox table]
  Outbox --> Worker[Celery 5.6.x worker]
  Worker --> Redis[(Redis 8.x broker)]
  Worker --> Mailpit[Mailpit 1.31.x]
  FastAPI --> OpenAPI[OpenAPI JSON and Swagger UI]
```

The FastAPI app is layered: routers validate HTTP contracts, services enforce ClinicQ rules, repositories isolate SQLAlchemy queries, and models map PostgreSQL tables. The booking service MUST use database constraints, row locks, or `SERIALIZABLE` isolation with retry; an ADR must record the chosen strategy. The report service MUST use hand-written SQL for the doctor performance report.

## Key flow sequences

### Patient registration and login

```mermaid
sequenceDiagram
  actor Patient
  participant API as FastAPI auth router
  participant Auth as Auth service
  participant DB as PostgreSQL
  Patient->>API: POST /api/v1/auth/register
  API->>Auth: validate e-mail, password, profile
  Auth->>DB: insert users and patients in one transaction
  DB-->>Auth: user_id, patient_id
  Auth-->>API: registered patient
  API-->>Patient: 201 patient profile
  Patient->>API: POST /api/v1/auth/login
  Auth->>DB: verify hash and recent failures
  Auth->>DB: insert refresh token hash
  API-->>Patient: 200 access token and refresh token
```

### Refresh rotation with reuse detection

```mermaid
sequenceDiagram
  actor User
  participant API as FastAPI auth router
  participant Auth as Token service
  participant DB as PostgreSQL
  User->>API: POST /api/v1/auth/refresh
  API->>Auth: presented refresh token
  Auth->>DB: lookup token_hash
  alt token already rotated
    Auth->>DB: revoke family_id
    API-->>User: 401 type refresh-token-reused
  else token revoked
    API-->>User: 401 type refresh-token-revoked
  else token valid
    Auth->>DB: set rotated_at and insert new token hash
    API-->>User: 200 new access and refresh token
  end
```

### Availability lookup and booking

```mermaid
sequenceDiagram
  actor Patient
  participant API as Appointment router
  participant Availability as Availability service
  participant Booking as Booking service
  participant DB as PostgreSQL
  Patient->>API: GET /api/v1/doctors/{id}/availability?date=2026-10-05
  Availability->>DB: read weekly sessions, leaves, active appointments
  Availability-->>API: free IST slots
  API-->>Patient: 200 slots
  Patient->>API: POST /api/v1/appointments
  Booking->>DB: begin transaction and lock or use serializable strategy
  Booking->>DB: enforce future, 30-day window, active uniqueness, patient limits
  Booking->>DB: insert appointment and audit row
  DB-->>Booking: committed BOOKED appointment
  API-->>Patient: 201 appointment
```

### Receptionist state change

```mermaid
sequenceDiagram
  actor Receptionist
  participant API as Appointment router
  participant Service as State machine service
  participant DB as PostgreSQL
  Receptionist->>API: POST /api/v1/appointments/{id}/check-in
  API->>Service: actor role and appointment ID
  Service->>DB: load appointment
  Service->>Service: require BOOKED to CHECKED_IN
  Service->>DB: update status and insert audit row
  API-->>Receptionist: 200 CHECKED_IN
```

### Reminder e-mail worker (Should)

```mermaid
sequenceDiagram
  participant Beat as Celery beat
  participant Worker as Celery worker
  participant DB as PostgreSQL
  participant Mailpit as Mailpit
  Beat->>Worker: due reminder task
  Worker->>DB: select due REMINDER_EMAIL outbox rows
  Worker->>DB: claim row by idempotency_key
  Worker->>DB: skip row when sent_at is already set
  Worker->>Mailpit: send reminder e-mail
  Worker->>DB: set sent_at or increment attempt_count
```

If SMTP succeeds and the worker crashes before `sent_at` is stored, a retry MAY send a duplicate e-mail. The outbox still keeps one reminder row per appointment.

## Data flow

1. HTTP requests enter through `/api/v1` routers with a request ID. Public health and auth endpoints skip bearer-token checks where documented.
2. Pydantic v2 request models validate JSON fields, ISO dates, UUIDs, pagination bounds, and enum values.
3. Auth dependencies verify JWT signatures, 15-minute expiry, active user state, and role permissions before service calls.
4. Services apply business rules from `BR-01` through `BR-31`. Appointment status changes always write `appointment_audit_log`.
5. SQLAlchemy repositories execute parameterized queries. The doctor performance report uses SQLAlchemy `text()` with bound parameters and a window function.
6. PostgreSQL stores timestamps in UTC. API response display fields use Asia/Kolkata offsets such as `2026-10-05T10:00:00+05:30`.
7. Errors leave the system as RFC 9457 Problem Details with stable `type` values.
8. JSON logs record request ID, method, path, status, duration, and authenticated user ID when present.

## Local deployment view

| Profile | Services | Resource target | Use |
|---|---|---:|---|
| Lite 8 GB | API, PostgreSQL, pytest runner | API 512 MB, PostgreSQL 512 MB | Mandatory Must-scope development and grading pre-check. Celery, Redis, Mailpit, and Locust are off. |
| Standard 16 GB | API, PostgreSQL, Redis, Celery worker, Mailpit, pytest runner, Locust | API 1 GB, PostgreSQL 1 GB, worker 512 MB | Full Must plus Should e-mail and performance testing. |

Docker Compose MUST pin images, including PostgreSQL 18.x, Redis 8.x, and Mailpit 1.31.x. A clean clone MUST start locally through documented `uv` and `docker compose` commands without paid services.

## Expected implementation repository tree

```text
clinicq-api/
  app/
    api/
      deps.py
      routers/
        auth.py
        doctors.py
        availability.py
        appointments.py
        reports.py
        health.py
    core/
      config.py
      logging.py
      security.py
      problem_details.py
    db/
      base.py
      session.py
      migrations/
    models/
    schemas/
    services/
      auth_service.py
      availability_service.py
      booking_service.py
      appointment_state_service.py
      report_service.py
      notification_service.py
    repositories/
    workers/
      celery_app.py
      reminder_tasks.py
  tests/
    unit/
    api/
    integration/
    concurrency/
    security/
    performance/
  docs/
    adr/
    query-plans/
  bruno/
  scripts/
    seed_demo.py
  docker-compose.yml
  docker-compose.lite.yml
  pyproject.toml
  uv.lock
  README.md
```

## Design principles

- Keep HTTP, service, repository, and model code separate so booking rules can be unit-tested without running FastAPI.
- Use PostgreSQL constraints and transactions for race-sensitive rules; do not rely only on Python in-memory checks.
- Make all configuration 12-factor environment variables with safe defaults in `.env.example`.
- Store only hashes for passwords and refresh tokens. Never log passwords, raw tokens, or Authorization headers.
- Prefer explicit Problem Details errors over framework-default JSON errors.
- Keep timestamps internally in UTC and convert to Asia/Kolkata only at API boundaries.
- Use an outbox table for notification side effects so appointment transactions do not depend on SMTP availability.

## Required ADR topics

| ADR topic | Exact question the student must answer |
|---|---|
| Booking transaction strategy | How does ClinicQ guarantee exactly one active appointment per doctor slot and patient limits under concurrent requests: partial unique indexes, row locks, `SERIALIZABLE` retry, or a combination? |
| Refresh token design | How are refresh tokens hashed, rotated, revoked on logout, and family-revoked when a rotated token is reused? |
| Availability query and indexes | Which query shape and indexes make availability fast for schedules, leaves, and active appointments, and what does `EXPLAIN ANALYZE` show? |
| Doctor performance report SQL | Why is the report implemented with hand-written SQL, how are parameters bound, and how is `revenue_rank` calculated with a window function? |
| Notification worker choice | If Should-scope e-mail is implemented, why are Celery, Redis, Mailpit, outbox rows, and idempotency keys used? |
| Deployment profile | Which Docker Compose profiles are supported on 8 GB and 16 GB hardware, and which services are disabled in the lite profile? |
| Optional technology choice | If async SQLAlchemy, Redis availability cache, Valkey, Schemathesis, or cloud deployment is chosen, what trade-off justifies it for ClinicQ? |

[Back to README](../README.md)
