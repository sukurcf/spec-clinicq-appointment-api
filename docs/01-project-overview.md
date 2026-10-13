# Project overview

## Problem

Entry-level data engineers need practice moving a small, inspectable file into a relational database without losing bad records or duplicating good ones. LoadLite makes that one batch path explicit and testable.

## Outcome

In exactly five days × six hours/day = 30 hours, produce a Python 3.12 CLI that reads one fictional `orders.csv` of at most 500 rows, validates its contract, writes rejected rows with reasons, and transactionally loads accepted orders into PostgreSQL 18.

The deterministic 20-row fixture has 18 accepted rows and 2 rejected rows. The first run leaves 18 target rows. Re-running the same file inserts 0 and leaves the target at 18 rows.

## Scope

| In scope | Out of scope |
|---|---|
| One CSV input, one `orders` table, one CLI | Additional datasets, databases, or runtime services |
| Field validation, quarantine, source duplicate handling | Hosted data, external runtime calls, or visualization |
| PostgreSQL transaction, stable key, upsert, exact summary | Streaming, scheduling, or distributed processing |
| Local Compose startup, health, persistence, and reset | Any non-local deployment |

## Success measures

- Valid rows are inserted once; later duplicates within the file are quarantined.
- A changed payload with an existing `order_id` updates one existing row.
- A database failure leaves no partial batch writes.
- The exact summary and quarantine match the fixture contract.
- All 20 specified tests pass; `loadlite` coverage reaches 80% line and 70% branch.

## Skill-to-job mapping

| Skill | LoadLite practice | Entry-level data-engineering task |
|---|---|---|
| Python file handling | Parse CSV and reject malformed input clearly | Inspect and validate inbound files |
| Data quality | Apply field rules and retain invalid rows with reasons | Profile and quarantine source records |
| Relational storage | Model a key and use a PostgreSQL transaction | Load a batch into a target table |
| Idempotency | Repeat an input and upsert changed values | Safely replay a scheduled load |
| Testing and operations | Verify counts, rollback, health, and persistence | Prove a local pipeline is reproducible |

[Back to README](../README.md)
