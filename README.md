# LoadLite — local CSV-to-PostgreSQL loader

LoadLite is a small command-line batch loader for entry-level data engineers. In **five days × six hours/day = 30 hours**, build a Python tool that validates one fictional CSV file, quarantines bad rows with reasons, and loads valid orders into one local PostgreSQL table.

## Quick facts

| Item | Value |
|---|---|
| Track | Entry-level data engineering |
| Audience | Freshers with basic Python, SQL, and Git |
| Duration | Exactly 5 days × 6 hours/day = 30 hours |
| Input | One `orders.csv`, at most 500 data rows |
| Stable key | `order_id` |
| Interface | One Python CLI |
| Runtime services | One local PostgreSQL 18 database |
| Fixture | 20 fictional rows: 18 accepted, 2 rejected |
| Required tests | 20 concrete cases; 80% line and 70% branch coverage for `loadlite` |

## Required outcome

- Validate `order_id`, `customer_id`, ISO `order_date`, and non-negative decimal `amount`.
- Write invalid rows to a quarantine CSV with a stable reason code.
- Keep the first valid source row for each `order_id`; quarantine later valid duplicates.
- Load all accepted rows in one transaction. A changed payload for an existing key updates that order; an unchanged file rerun creates no rows.
- Print exactly five JSON counts: `source`, `accepted`, `rejected`, `inserted`, and `already_present`.

First load of the fixture:

```json
{"source":20,"accepted":18,"rejected":2,"inserted":18,"already_present":0}
```

On the same-file rerun, `inserted` is `0`, `already_present` is `18`, and the target still has **18 rows**.

## Local demo

After the one-time setup in [06 Tech stack and setup](docs/06-tech-stack-and-setup.md), start PostgreSQL, confirm its health check, and run:

```bash
uv run --env-file .env.local --offline loadlite run fixtures/orders.csv --format json --quarantine .local/quarantine/orders-rejected.csv
```

The database is published only at `127.0.0.1:25432`. Stop and restart preserve its named volume. Runtime needs no external connection after the first dependency and image downloads.

## Scope

The deliverable is a local Python CSV-to-PostgreSQL loader with a deterministic fixture, useful tests, and reproducible local operations. It does not require any other runtime service, hosted resource, external data source, scheduler, queue, or visualization. The specification repository contains Markdown only; students implement the loader in their own repository.

## Read the specification

| Document | Purpose |
|---|---|
| [01 Project overview](docs/01-project-overview.md) | Problem, outcome, scope, and job skills |
| [02 Users and roles](docs/02-users-and-roles.md) | Two roles and three user stories |
| [03 Functional requirements](docs/03-functional-requirements.md) | Five Must requirements and data rules |
| [04 Non-functional requirements](docs/04-non-functional-requirements.md) | Performance, reliability, security, and hardware |
| [05 System architecture](docs/05-system-architecture.md) | One architecture diagram and data flow |
| [06 Tech stack and setup](docs/06-tech-stack-and-setup.md) | Versions, local commands, port, fixture, and lifecycle |
| [07 Data model](docs/07-data-model.md) | CSV, target table, quarantine, and fixture contract |
| [08 CLI specification](docs/08-api-specification.md) | Exact commands, errors, and JSON counts |
| [09 Testing strategy and test cases](docs/09-testing-strategy-and-test-cases.md) | 20 distinct tests and traceability |
| [10 Development and CI quality](docs/10-devops-ci-cd-and-quality.md) | Git, CI, environment, and Definition of Done |
| [11 Five-day plan](docs/11-milestones-and-deliverables.md) | 30-hour schedule and five-minute demo |
| [12 Evaluation rubric](docs/12-evaluation-rubric.md) | Fixed 100-point grading |
| [13 Interview preparation](docs/13-interview-preparation.md) | Ten role-specific questions |
| [14 Glossary and resources](docs/14-glossary-and-resources.md) | Terms and official learning links |

## Learning and integrity

Use assistants to explain concepts or review errors, but understand and verify every submitted change. Follow your trainer’s rules for disclosing assistance. Use only fictional data and keep local credentials out of Git.

## Change log

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-10-02 | Initial specification release. |
| 1.1 | 2026-10-02 | Local-only, backend-focused revision; the original six-week duration and rubric weights were unchanged at that revision. |
| 1.2 | 2026-10-13 | Replaces the earlier scope with LoadLite and exactly five days × six hours/day = 30 hours. Grading now measures data quality, idempotent transactional loading, tests, and local reproducibility; earlier scope and optional work do not earn marks. |
