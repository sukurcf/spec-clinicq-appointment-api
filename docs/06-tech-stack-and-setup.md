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
| Process manager | Gunicorn | 26.x | Optional production-style runner | Useful for interview discussion, not required for local Must scope. |
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
| Local-only demo | Free cloud demo | What is the cost warning and destroy-after-demo step? |

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

## Local setup checklist

### Windows 11 with WSL2 Ubuntu 24.04

1. Enable WSL2 and install Ubuntu 24.04.
2. Install Docker Desktop with the WSL2 backend.
3. Apply the `.wslconfig` values from this document.
4. Install Git inside WSL2.
5. Install uv inside WSL2.
6. Install VS Code and the WSL, Python, Docker, Ruff, and Markdown extensions.
7. Clone the student repository `clinicq-api` inside the WSL filesystem.
8. Start PostgreSQL and the API with Docker Compose.
9. Run migrations, seed data, tests, and the Bruno collection.
10. Keep all demo e-mail addresses on `example.in` or `example.com`.

### macOS

1. Install Git, uv, and Docker Desktop or a compatible container runtime.
2. Install VS Code with Python, Docker, Ruff, and Markdown extensions.
3. Clone `clinicq-api`.
4. Start services with Docker Compose.
5. Run migrations, seed data, pytest, and the Bruno collection.

### Linux

1. Install Git, uv, Docker Engine, and Docker Compose.
2. Add the user to the Docker group according to official Docker docs.
3. Install VS Code or another editor with Python support.
4. Clone `clinicq-api`.
5. Start services with Docker Compose.
6. Run migrations, seed data, tests, and the Bruno collection.

## Required `.wslconfig` values

Use these values unless the trainer writes a reason in the project notes.

```text
8 GB laptop:
memory=4GB
swap=4GB

16 GB laptop:
memory=8GB
swap=4GB
```

## Hardware profiles

| Profile | Laptop | Services on | Services off | Exact limits |
|---|---|---|---|---|
| Lite | 8 GB RAM | API, PostgreSQL, test runner | Celery, Redis, Mailpit, Locust, cloud demo | API 512 MB; PostgreSQL 512 MB; no worker process during Must tests. |
| Standard | 16 GB RAM | API, PostgreSQL, Redis, Celery, Mailpit, test runner | Cloud demo unless selected | API 1 GB; PostgreSQL 1 GB; worker 512 MB; Redis 256 MB; Mailpit 256 MB. |

The Must scope MUST pass on the lite profile. Should e-mail and reminder work may use the standard profile.

## Trainer pre-check

- Start the lite profile on one 8 GB laptop before week 1.
- Run the clean-clone setup from the student README.
- Confirm PostgreSQL readiness returns success.
- Run the seed command for 5 doctors and 20 patients.
- Run the concurrency tests for FR-BOOK-02.
- Run the Bruno or Postman smoke collection.
- Confirm no service requires paid cloud access.
- Record any laptop-specific limits in the cohort notes.

## Free accounts needed

| Account | Required? | Use |
|---|---|---|
| GitHub | Yes | Student repository, issues, PRs, and Actions. |
| Docker account | No for local work | Only needed if Docker Desktop asks for sign-in. |
| Bruno account | No | Local collection can run without cloud sync. |
| Cloud provider | No | Could-scope demo only, with cost warning. |

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
| 8 | Docker Compose, health checks, logs | 3 | Local demo and operations. |
| 9 | Celery, Redis, Mailpit basics | 2 | Should-scope notifications. |
| 10 | Locust and EXPLAIN ANALYZE basics | 1 | Performance evidence. |

Total guided learning and setup time is about 40 hours. About 30 hours count inside Must scope during week 1 and early week 2.

[Back to README](../README.md)
