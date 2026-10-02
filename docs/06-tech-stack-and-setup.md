# Tech stack and setup

Purpose: This document defines the ClinicQ toolchain, reference versions, local setup, hardware profiles, and learning order.

## Reference versions and mandatory stack

Use these minor lines. Do not use `latest` image tags. Commit lockfiles.

| Category | Tool | Reference version | Purpose | Why ClinicQ uses it |
|---|---|---:|---|---|
| Language | Python | 3.12.x | Backend language | Current cohort baseline with broad FastAPI support. |
| Package manager | uv | 0.12.x | Create environments and lock dependencies | Fast installs and reproducible `uv.lock`. |
| API framework | FastAPI | 0.142.x | REST API and OpenAPI | Good fit for `/api/v1`, validation, and docs. |
| Validation | Pydantic | 2.13.x | Request and response models | Clear schemas for appointments and Problem Details. |
| Settings | pydantic-settings | 2.15.x | Environment configuration | Keeps secrets out of code. |
| ORM | SQLAlchemy | 2.1.x | Database access | Supports 2.0-style ORM and explicit transactions. |
| Migrations | Alembic | 1.20.x | Schema migration history | Required for tables and indexes. |
| Database | PostgreSQL | 18.x | Primary datastore | Supports partial unique indexes and transactions. |
| Driver | psycopg | 3.3.x | PostgreSQL connection | Modern PostgreSQL driver for SQLAlchemy. |
| API server | Uvicorn | 0.54.x | Local ASGI server | Runs FastAPI during development. |
| Process manager | Gunicorn | 26.x | Optional LOCAL multi-worker runner | May illustrate database concurrency and process management; no deployment task is required. |
| Auth library | PyJWT | 2.15.x | JWT creation and verification | Implements short access tokens. |
| Password hashing | pwdlib | 0.3.x | Argon2 or bcrypt helpers | Avoids raw password storage. |
| Background jobs | Celery | 5.6.x | Should-scope reminders | Explains worker retries and idempotency. |
| Queue | Redis server / redis-py | 8.x / 8.1.x | Celery broker in Should scope | Local and common in backend roles. |
| Containers | Docker Engine | 29.x | Run services locally | One-command reproducible demos. |
| Compose | Docker Compose | v5.x | Start API, database, Mailpit, Redis, worker | Matches the project service list. |
| E-mail sandbox | Mailpit | 1.31.x | Local e-mail viewing | MailHog is not allowed. |
| Test runner | pytest | 9.1.x | Unit, API, and integration tests | Required for all Must rules. |
| Coverage | pytest-cov / coverage | 7.1.x / 7.16.x | Coverage gates | Enforces 85% line and 75% branch coverage. |
| HTTP tests | httpx / TestClient | 0.28.x / FastAPI TestClient | API tests | Exercises real HTTP behavior. |
| Integration DB | Testcontainers Python | 4.15.x | PostgreSQL tests | Verifies migrations and concurrency on PostgreSQL. |
| Test data | factory_boy or polyfactory | 3.3.x / 3.3.x | Create fixture data | Keeps patient and doctor examples readable. |
| Load testing | Locust | 2.46.x | Should-scope performance test | Checks availability p95 at 50 users. |
| Contract tests | Schemathesis | 4.29.x | Could-scope OpenAPI testing | Finds mismatches between docs and API. |
| Lint and format | Ruff | 0.16.x | Style and formatting | Fast checks for Python code. |
| Typing | mypy | 2.4.x | Static type checking | Catches service and DTO mistakes early. |
| Hooks | pre-commit | 4.6.x | Local quality gates | Runs checks before commits. |
| Dependency audit | pip-audit | 2.10.x | Python vulnerability checks | Supports CI security gate. |
| Image scan | Trivy | 0.75.x | Container vulnerability checks | Checks the API image. |
| Secret scan | Gitleaks | 8.30.x | Detect committed secrets | Mandatory no-secret gate. |
| API client | Bruno | Current official app | Demo API collection | Recommended collection format for local demos. |
| Editor | VS Code | Current stable | Student editor | Good Python, Docker, and REST extensions. |
| Version control | Git | Current stable | Commits and branches | Required for GitHub Flow. |

## Allowed alternatives

Each alternative needs an ADR before implementation.

| Default | Allowed alternative | ADR question |
|---|---|---|
| Argon2 through pwdlib | bcrypt through pwdlib or Passlib | Which password hash is used, and why is it safe? |
| Bruno collection | Postman collection | How will the demo collection remain shareable without secrets? |
| SQLAlchemy synchronous session | SQLAlchemy async session | Does async reduce complexity for this student project? |
| Redis server | Valkey 9.x | Is the broker API compatible with Celery reminders? |
| factory_boy | polyfactory | Which fixture style keeps booking tests clearer? |

## Not allowed

| Tool or practice | Reason |
|---|---|
| `latest` Docker image tags | Demos become non-reproducible. |
| SQLite for Must scope | It does not prove PostgreSQL locking and partial indexes. |
| MailHog | It is unmaintained; use Mailpit. |
| Bitnami images or charts | The free catalog changed in 2025. |
| Real clinic data | It violates privacy and the project rules. |
| In-memory booking locks only | They fail with multiple API processes. |
| Exactly-once e-mail claims | E-mail is external; use at-least-once with idempotency. |
| Direct push to `main` | The project uses PR review and CI gates. |
| Student-built interfaces or public cloud exercises | Backend/local-only scope applies to Must, Should, Could, and bonus work. |

## Local setup checklist

### Windows 11 with WSL2 Ubuntu 24.04

1. Enable WSL2 and install Ubuntu 24.04.
2. Install Docker Desktop with the WSL2 backend.
3. Apply the `.wslconfig` values from this document.
4. Install Git inside WSL2.
5. Install uv inside WSL2.
6. Install VS Code and the WSL, Python, Docker, Ruff, and Markdown extensions.
7. Clone the student repository `clinicq-api` inside the WSL filesystem.
8. Use the single local start entry point below; it initializes migrations and fictional fixtures.
9. Run health checks, the deterministic API operation, pytest, and the stop entry point.
10. Keep all demo e-mail addresses on `example.in` or `example.com`.

### macOS

1. Install Git, uv, and Docker Desktop or a compatible container runtime.
2. Install VS Code with Python, Docker, Ruff, and Markdown extensions.
3. Clone `clinicq-api`.
4. Use the same local start entry point and hardware budget below.
5. Run health/demo checks, pytest, and the documented stop entry point.

### Linux

1. Install Git, uv, Docker Engine, and Docker Compose.
2. Add the user to the Docker group according to official Docker docs.
3. Install VS Code or another editor with Python support.
4. Clone `clinicq-api`.
5. Use the same local start entry point below.
6. Run health/demo checks, tests, and the documented stop entry point.

Python 3.12 and `curl` are required on the host for the Python test driver and health commands. The commands below describe deliverables students MUST implement in their application repository, not runnable files in this Markdown-only specification repository.

## Required `.wslconfig` values

Use these values unless the trainer writes a reason in the project notes.

| Host RAM | WSL memory | WSL swap |
|---|---|---|
| 8 GB | 4 GB | 4 GB |
| 16 GB | 8 GB | 4 GB |

## Local operation contract

### Downloads versus local runtime

Initial cloning and downloads of pinned images, test-helper images, Python 3.12, and `uv.lock` packages need internet. GitHub submission and hosted CI need internet too. After downloads, startup, API demonstrations, and runtime tests MUST use local caches and recorded fictional fixtures without external network access. No paid account, public hostname, API key, or cloud resource is required. Local verification must not depend on GitHub or a registry.

### Single start and stop entry points

From the implementation repository root:

```bash
sh scripts/local-start.sh --profile lite
sh scripts/local-stop.sh
```

`--profile standard` uses the same ports/data contract with the larger resource budget. WSL2 users run inside Ubuntu; macOS/Linux users use the POSIX shell.

Students MUST implement start to check Docker and occupied ports, generate untracked local secrets/configuration on first use, prepare host `uv sync --frozen` and locked API-image dependencies during initial downloads, and start Compose. Cached restarts use existing images without a build or registry pull. Missing offline caches cause nonzero `local-start: required image or dependency is not cached`, not a silent external fetch.

After PostgreSQL is ready, start MUST run `uv run alembic upgrade head` and `uv run python scripts/seed_demo.py --fixture local` inside the API container. Seed creates missing fictional records but never overwrites edits, deletes appointments/audit rows, or rotates credentials/signing secrets on restart. Start waits for API readiness and prints `ClinicQ ready at http://127.0.0.1:8001`. DB timeout exits nonzero with `local-start: PostgreSQL unavailable at 127.0.0.1:25432`; an occupied API port reports `local-start: port 8001 is already in use`.

Stop shuts down only this Compose project without `--volumes` and prints `ClinicQ stopped; local data preserved`. It preserves named volumes and untracked local configuration.

### Fixed loopback ports

| Service | Host binding | Container address | Default |
|---|---|---|---|
| FastAPI, health, OpenAPI | `127.0.0.1:8001` | `api:8000` | On |
| PostgreSQL | `127.0.0.1:25432` | `db:5432` | On |
| Redis | `127.0.0.1:16379` | `redis:6379` | Optional LOCAL reminder/cache work |
| Mailpit SMTP | `127.0.0.1:21025` | `mailpit:1025` | Optional LOCAL e-mail work |
| Mailpit built-in console | `127.0.0.1:28025` | `mailpit:8025` | Optional operations inspection only |

Host-published bindings MUST be `127.0.0.1`, never `0.0.0.0`. Container servers may listen on container interfaces. Celery exposes no host port. These ports do not conflict with CampusHire's documented local profile.

### Persistent data and explicit reset

Named volume `clinicq_pgdata` persists users, password hashes, appointments, booking-time fees, audit logs, token families, and outbox rows. Optional `clinicq_redisdata` persists the local queue; PostgreSQL remains authoritative. Keep generated secrets in untracked `.local/local.env`, credentials in `.local/demo-credentials.json`, and first-seed date metadata in `.local/demo-context.json`. Mount local configuration read-only into API/worker containers and retain it across stop/start.

Default `LOCAL_INSTANCE=clinicq` selects these paths/volumes. Acceptance uses `LOCAL_INSTANCE=clinicq_acceptance`, separate `clinicq_acceptance_pgdata`/`clinicq_acceptance_redisdata`, and files under `.local/acceptance/`. Only one ClinicQ instance may use the fixed ports at a time; the host test driver must not interfere with an unrelated running stack.

The separate destructive reset command is:

```bash
sh scripts/local-reset.sh --confirm-delete-local-data
```

Without that flag, reset exits nonzero with `local-reset: explicit confirmation required` and leaves all records/configuration intact. With confirmation, it removes only the selected project's local volumes, generated secrets, credentials, and date context. It must not remove other projects' data. A later start rebuilds migrations/seed, not remote resources.

### Fictional seed and recorded fixtures

The first seed creates exactly 5 doctors, 20 patients, two weeks of demo schedules, at least 100 appointments, and local staff accounts for Rohan, Neha, and Iqbal. Commit fictional fixture templates under `fixtures/local-demo/` in the implementation repository and record their SHA-256. No fixture, e-mail check, or runtime test downloads external data.

Preserve existing aliases: Rohan `D-100` is UUID `55555555-5555-4555-8555-555555555555`; Asha's patient UUID is `22222222-2222-4222-8222-222222222222`. Rohan's local-demo Monday fixture has one `09:00-10:00` IST session and fee `600.00`. Asha initially has no upcoming active appointment. Reserve Rohan's four slots on `demo_date` with no leave/active booking; the 100 sample appointments use other dates/doctors or historical terminal states.

At first seed, `demo_date` is the next Monday strictly after the seed's current IST date, within seven days. Persist `seed_as_of` and `demo_date` in the local context; normal restart must not shift booked dates or overwrite appointments. Manual demo requests use that saved date. Fixed clocks remain restricted to automated tests: a host acceptance driver may set `APP_ENV=test` and `TEST_BUSINESS_NOW=2026-10-02T10:00:00+05:30`, yielding `demo_date=2026-10-05`. Non-test settings MUST reject that override; JWT/refresh expiry and login security use real time except in explicit security tests.

Optional Celery/Redis/Mailpit run locally only. E-mail tests inspect captured Mailpit messages/outbox rows; built-in consoles are not a required business interface. Live integrations are outside the local deliverables and never automatic test fallbacks.

### Exact health checks and deterministic API operation

```bash
curl --fail --silent -H 'X-Request-ID: req-local-live' http://127.0.0.1:8001/api/v1/health/live
curl --fail --silent http://127.0.0.1:8001/api/v1/health/ready
```

Expected status/body pairs are `200 {"status":"live","request_id":"req-local-live"}` and `200 {"status":"ready","database":"ok"}`. With DB stopped, liveness stays `200`; readiness returns `503` Problem Details type `database-not-ready`, never a raw driver exception.

For the deterministic core operation, login as seeded Asha using generated local credentials, keep the bearer token, and GET `/api/v1/doctors/55555555-5555-4555-8555-555555555555/availability?date=<demo_date>` before creating a demo booking. Expected status is `200`, with the exact shape below. Replace every `2026-10-05` with the saved date for a manual run; the controlled acceptance fixture matches this JSON literally:

```json
{
  "doctor_id": "55555555-5555-4555-8555-555555555555",
  "date": "2026-10-05",
  "time_zone": "Asia/Kolkata",
  "slot_length_minutes": 15,
  "slots": [
    {"starts_at":"2026-10-05T09:00:00+05:30","ends_at":"2026-10-05T09:15:00+05:30"},
    {"starts_at":"2026-10-05T09:15:00+05:30","ends_at":"2026-10-05T09:30:00+05:30"},
    {"starts_at":"2026-10-05T09:30:00+05:30","ends_at":"2026-10-05T09:45:00+05:30"},
    {"starts_at":"2026-10-05T09:45:00+05:30","ends_at":"2026-10-05T10:00:00+05:30"}
  ]
}
```

Run the student-implemented host acceptance driver, not a test container that stop would terminate:

```bash
uv run --offline pytest -q tests/local/ --junitxml=docs/test-reports/local.xml
```

It caches prerequisites before testing, disables external egress while retaining loopback/Compose networking, and runs TC-LOCAL-001 through TC-LOCAL-006 in document 09. Include the pinned PostgreSQL/test-helper images in the initial cache so Testcontainers does not pull during the offline phase.

The host driver runs unit/API/database/concurrency/security runtime selections without recursively invoking `tests/local/`. It remains alive during API-container shutdown and validates persisted state before rotating the old refresh token.

## Hardware profiles

| Profile | Laptop | Services on | Services off | Exact limits |
|---|---|---|---|---|
| Lite | 8 GB RAM, four cores, 20 GB free disk | API, PostgreSQL, one Python test worker | Local Celery, Redis, Mailpit, Locust unless testing selected Should work | API 512 MB; PostgreSQL 512 MB; Docker memory 4 GB, swap 4 GB. |
| Standard | 16 GB RAM, four cores, 20 GB free disk | API, PostgreSQL, Python tests; optional local Redis/Celery/Mailpit | Optional services when not selected | API 1 GB; PostgreSQL 1 GB; worker 512 MB; Redis 256 MB; Mailpit 256 MB; Docker memory 8 GB, swap 4 GB. |

The Must scope MUST pass on the lite profile. Should local e-mail/reminder work may use the standard profile. These are proposed resource budgets, not measured results.

## Trainer pre-check

- Start the lite profile on an 8 GB, four-core laptop with 20 GB free disk before week 1.
- Run the clean-clone setup from the student README.
- Confirm both exact health status/body pairs below and the deterministic local API operation.
- Run the seed command for 5 doctors and 20 patients.
- Run the concurrency tests for FR-BOOK-02.
- Run the local API collection or Python HTTP demonstration.
- Disable external egress after downloads and run TC-LOCAL-002 plus the runtime suite from recorded fixtures.
- Run TC-LOCAL-003 and compare appointment/audit/token IDs and states after stop/start.
- Confirm invalid input, unavailable PostgreSQL, missing offline caches, occupied ports, loopback binding, and reset confirmation behavior.
- Confirm no service requires paid cloud access.
- Record any laptop-specific limits in the cohort notes.

## Free accounts needed

| Account | Required? | Use |
|---|---|---|
| GitHub | Yes | Student repository, issues, PRs, and Actions. |
| Docker account | No for local work | Only needed if Docker Desktop asks for sign-in. |
| Bruno account | No | Local collection can run without cloud sync. |

## Suggested learning order

| Order | Topic | Hours | Related ClinicQ work |
|---:|---|---:|---|
| 1 | GitHub Flow, issues, PRs, Conventional Commits | 3 | Repository setup and first PR. |
| 2 | Python 3.12, uv, project structure | 4 | Application skeleton. |
| 3 | FastAPI, Pydantic, OpenAPI, HTTP status codes | 6 | `/api/v1` contracts. |
| 4 | PostgreSQL, SQLAlchemy, Alembic | 6 | Doctor, patient, and appointment data. |
| 5 | JWT, refresh tokens, password hashing, RBAC | 5 | FR-AUTH-01 to FR-AUTH-04. |
| 6 | Transactions, isolation, locks, indexes | 5 | FR-BOOK-02 and reports. |
| 7 | pytest, Testcontainers, coverage | 5 | Test catalog and CI gates. |
| 8 | Docker Compose, health checks, logs, persistence, offline fixtures | 3 | Clean start/stop, seed idempotency, local error and durability acceptance. |
| 9 | Celery, Redis, Mailpit basics | 2 | Should-scope notifications. |
| 10 | Locust and EXPLAIN ANALYZE basics | 1 | Performance evidence. |

Total guided learning and setup time is about 40 hours. About 30 hours count inside Must scope during week 1 and early week 2.

Version 1.1 assigns removed future-client/deployment work to Python/SQL/API contracts, fixture regressions, and local operations. No interface or public deployment earns bonus points.

[Back to README](../README.md)
