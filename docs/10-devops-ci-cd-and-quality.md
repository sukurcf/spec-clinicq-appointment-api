# DevOps, CI/CD, and quality

Purpose: This document defines ClinicQ Git workflow, code quality, CI gates, Docker requirements, configuration, releases, and Definition of Done.

## Git workflow

ClinicQ uses GitHub Flow.

1. Create a GitHub Issue for each small task.
2. Create one branch from `main` for that issue.
3. Commit with Conventional Commits.
4. Open a pull request before merging.
5. Run CI on every push and PR.
6. Merge only after CI is green and the PR checklist is complete.

## Branch naming

| Work type | Pattern | Example |
|---|---|---|
| Feature | `feature/<issue>-<short-name>` | `feature/12-book-appointment` |
| Bug fix | `fix/<issue>-<short-name>` | `fix/31-refresh-reuse` |
| Test work | `test/<issue>-<short-name>` | `test/44-same-slot-race` |
| Documentation | `docs/<issue>-<short-name>` | `docs/52-explain-analyze` |
| Chore | `chore/<issue>-<short-name>` | `chore/60-update-ruff` |

## Conventional Commits

| Type | ClinicQ example |
|---|---|
| `feat` | `feat: add patient booking endpoint` |
| `fix` | `fix: reject duplicate active doctor slots` |
| `test` | `test: cover refresh token reuse` |
| `docs` | `docs: record booking transaction ADR` |
| `refactor` | `refactor: split appointment service rules` |
| `chore` | `chore: pin postgres 18 image` |
| `ci` | `ci: add coverage gate` |

Every commit SHOULD reference the issue in the body, for example `Refs #12`.

## Pull request checklist

- The PR changes one feature area only.
- Linked issue is present.
- Must FR IDs are named when applicable.
- Tests cover success, negative, and authorization paths.
- Booking changes include race-condition tests.
- API changes update OpenAPI examples and the selected local collection or Python HTTP demonstration.
- Database changes include an Alembic migration.
- New indexes have EXPLAIN ANALYZE notes when they affect availability or reports.
- No raw password, token, or Authorization header is logged.
- `uv.lock` is updated when dependencies change.
- The PR description mentions significant AI help, if used.
- Local-operation changes include fixture, offline, persistence, and actionable-error evidence from document 09.

## Branch protection

| Rule | Requirement |
|---|---|
| Protected branch | `main` |
| Required checks | lint, type check, tests/coverage, Docker build, security scans, local lifecycle |
| Direct push | Not allowed |
| Required PR | Yes |
| Stale approval | Dismiss when new commits arrive |
| Conversation resolution | Required before merge |
| Trainer review | At least 2 substantive PRs each week are reviewed by trainer. |

## Code-quality tools and rules

| Tool | Required rule |
|---|---|
| Ruff | Lint and format application, tests, and scripts. |
| mypy | Check application packages with `disallow_untyped_defs` and `no_implicit_optional`. |
| pytest | Run at least 50 meaningful cases, all Must catalog rows, and unit/API/database/concurrency/security/report/local tests. |
| pytest-cov | Fail below 85% total line or 75% total branch coverage. |
| targeted coverage | Booking, slot generation, and state machine are at least 95% line and 85% branch coverage. |
| pre-commit | Run Ruff, mypy, whitespace, end-of-file, and Gitleaks hooks. |
| pip-audit | Fail on fixable high or critical Python dependency vulnerabilities. |
| Trivy | Scan the API image and fail on high or critical unfixed issues. |
| Gitleaks | Fail when secrets are detected in working tree or history scan. |

## CI pipeline table

| Job | Trigger | Steps in words | Gate or failure condition |
|---|---|---|---|
| Lint | PR and push to `main` | Install locked dependencies and run Ruff check plus format check. | Fails on lint or format differences. |
| Type check | PR and push to `main` | Run mypy on application packages. | Fails on type errors. |
| Unit tests | PR and push to `main` | Run fast service tests for auth, booking rules, slots, and state machine. | Fails on any failing test. |
| API and integration tests | PR and push to `main` | Start PostgreSQL, migrate, load recorded fictional fixtures, and test endpoints plus the Python core-flow smoke. | Fails on broken endpoints/migrations or fewer than 50 meaningful cases. |
| Concurrency tests | PR and push to `main` | Run same-slot and patient-limit parallel booking tests. | Fails unless expected 201 and 409 counts match. |
| Coverage | PR and push to `main` | Combine test coverage reports and enforce each scope separately. | Fails below total 85% line, total 75% branch, service 95% line, service 85% branch, auth 90% line, auth 80% branch, report 85% line, or report 75% branch. |
| Docker build | PR and push to `main` | Build the API image with pinned base image. | Fails if image cannot build. |
| Security scan | PR and push to `main` | Run pip-audit, Trivy, and Gitleaks. | Fails on configured vulnerability or secret findings. |
| API smoke | PR and push to `main` and before demo | Use Python HTTP assertions for auth, search, availability, booking, conflict, report, and health; the local client collection is an alternative demo artifact. | Fails on any core contract mismatch; no client build toolchain is required. |
| Local lifecycle | PR and push to `main` | Cache prerequisites, then run host-side TC-LOCAL-001 through TC-LOCAL-006 with external egress disabled and isolated acceptance data. | Fails on startup, exact fixture/health mismatch, persistence loss, unclear errors, non-loopback bindings, or unsafe reset; save JUnit/config-hash evidence. |

Hosted CI and initial dependency/image/security-database downloads need internet. Local startup must not depend on GitHub or a registry; runtime acceptance follows downloads and uses only loopback/Compose networking. Online security scanning is separate from the offline runtime phase. No public-cloud deployment job or frontend build exists.

## Docker and Compose requirements

| Service | Must or Should | Requirement |
|---|---|---|
| API | Must | Runs FastAPI with environment variables and JSON logs. |
| PostgreSQL | Must | Uses pinned PostgreSQL 18.x image and persistent local volume. |
| Mailpit | Should | Runs only when e-mail features are tested or demoed. |
| Redis | Should | Runs only for Celery reminder work. |
| Celery worker | Should | Uses the same application image and idempotency keys. |
| Test runner | Must | Runs pytest against isolated test database. |

Compose MUST support a lite profile. The lite profile starts API and PostgreSQL only.

Students MUST implement document 06's start/stop entry points, automatic migration plus non-destructive seed initialization, fixed loopback ports, and explicit-confirmation reset. API host port is `8001`, PostgreSQL `25432`, optional Redis `16379`, optional Mailpit SMTP `21025`, and its built-in console `28025`. Named volumes/configuration survive stop/start; only the selected project's confirmed reset deletes them. API/worker images and test-helper images must already be cached for offline runs.

## Environment variables

| Name | Example value | Purpose | Secret? |
|---|---|---|---|
| `APP_ENV` | `local` | Select local behavior. | No |
| `API_PREFIX` | `/api/v1` | Versioned API prefix. | No |
| `DATABASE_URL` | `postgresql+psycopg://clinicq:<password>@db:5432/clinicq` | PostgreSQL connection. | Yes in real deployments |
| `JWT_SECRET_KEY` | `change-me-local-only` | Sign access tokens. | Yes |
| `ACCESS_TOKEN_MINUTES` | `15` | Access token lifetime. | No |
| `REFRESH_TOKEN_DAYS` | `7` | Refresh token lifetime. | No |
| `PASSWORD_HASH_SCHEME` | `argon2` | Selected password hashing scheme. | No |
| `LOCKOUT_FAILURES` | `5` | Login failure threshold. | No |
| `LOCKOUT_WINDOW_MINUTES` | `15` | Failure counting window. | No |
| `LOCKOUT_MINUTES` | `15` | Account lock duration. | No |
| `CLINIC_TIME_ZONE` | `Asia/Kolkata` | Display and local-date rules. | No |
| `MAIL_SMTP_HOST` | `mailpit` | Mailpit SMTP host. | No |
| `MAIL_SMTP_PORT` | `1025` | Mailpit SMTP port. | No |
| `REDIS_URL` | `redis://redis:6379/0` | Celery broker in Should scope. | No |
| `LOCAL_INSTANCE` | `clinicq` | Select project-scoped volumes/config; acceptance uses a disposable isolated instance. | No |
| `TEST_BUSINESS_NOW` | `2026-10-02T10:00:00+05:30` | Business clock override only with `APP_ENV=test`; ordinary local runs reject it. | No |
| `LOG_LEVEL` | `INFO` | JSON log verbosity. | No |
| `REQUEST_ID_HEADER` | `X-Request-ID` | Correlation ID header. | No |

The student repository MUST include `.env.example` with placeholder values only.

The start entry point generates untracked local signing/DB secrets once and retains them on restart. JWT/refresh expiry and lockout use real time except explicit auth-test clocks. Saved local demo dates anchor at first seed and do not shift existing appointments on restart.

## Versioning and release

- Use SemVer for student releases.
- Tag the final demo as `v1.0.0`.
- Maintain a CHANGELOG in the student repository.
- Record ADRs for transaction strategy, token strategy, raw SQL report, optional local e-mail worker, and local lifecycle/persistence.
- Never tag a release when CI is red.

## Dependency management

| Rule | Requirement |
|---|---|
| Lockfile | Commit `uv.lock`. |
| Dependency updates | Use small PRs and rerun all tests. |
| Dependabot | MAY open dependency PRs. |
| Major upgrades | Require an ADR when they affect framework, database, or worker behavior. |
| Vulnerability fixes | Prioritize security PRs before Should features. |
| Docker tags | Pin minor or patch tags. Never use `latest`. |

## Definition of Done

A ClinicQ task is done only when all applicable items are true.

- The linked FR or NFR is named in the PR.
- The implementation matches the exact business rules.
- Tests prove positive, negative, and authorization behavior.
- Booking changes pass concurrency tests.
- Migrations run on empty and seeded databases.
- OpenAPI and the selected API collection/Python HTTP demo match endpoint behavior.
- Logs include request ID and omit secrets.
- CI is green on the branch.
- The PR checklist is complete.
- The student can explain the change during Friday demo.
- Applicable offline-fixture, persisted stop/start, loopback, dependency-error, and reset cases pass with sanitized evidence.

[Back to README](../README.md)
