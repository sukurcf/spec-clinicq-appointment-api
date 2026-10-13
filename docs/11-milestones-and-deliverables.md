# Five-day plan and deliverables

The complete effort is exactly **five days × six hours/day = 30 hours**, including setup, tests, local run, documentation, and demo.

## Plan

| Day | Hours | Work and linked requirements | Deliverable |
|---|---:|---|---|
| Day 1 | 6 | Read the brief, install Python 3.12/uv/Docker, inspect the fixture and contract, create the package outline and first test. | Local tools ready; first test and thin CSV slice. |
| Day 2 | 6 | Implement exact header, UTF-8/row-limit checks, field validation, and quarantine reasons. (FR-LOAD-01, FR-LOAD-02) | Deterministic parser and validation tests. |
| Day 3 | 6 | Define the one-table model; connect with psycopg; implement first-valid-key handling, transactional insert, changed-payload upsert, and rollback. (FR-LOAD-02, FR-LOAD-03) | PostgreSQL load path and integration tests. |
| Day 4 | 6 | Implement exact JSON counts, same-file replay, local health/errors, persistence, and confirmed reset. Finish all tests and meet coverage. (FR-LOAD-03–05) | 20 passing test cases; 80% line and 70% branch coverage for `loadlite`. |
| Day 5 | 6 | Run from a clean implementation checkout with external access disabled after downloads; fix defects; complete docs and a five-minute demo. | Reproducible local run and final explanation. |
| **Total** | **30** |  |  |

## Submission checklist

- Python 3.12 CLI and locked dependencies.
- Local Compose configuration for PostgreSQL 18 only, pinned image and loopback port.
- Fictional 20-row `orders.csv`; no fixture or implementation file is added to this specification repository.
- Quarantine with reason codes; one transaction; exact summary counts.
- All 20 test cases and the required package coverage.
- Startup, health, stop/start persistence, offline runtime, and explicit reset instructions.
- A short README and a five-minute demonstration.

## Five-minute demo

| Minute | Show |
|---:|---|
| 0–1 | Start the local database and confirm Compose health on the documented port. |
| 1–2 | Run the fixture and show `20 / 18 / 2 / 18 / 0`, 18 target rows, and two quarantine reasons. |
| 2–3 | Rerun the same file; show `inserted: 0`, `already_present: 18`, and 18 target rows. |
| 3–4 | Show the changed-payload upsert and transactional rollback test results. |
| 4–5 | Show all 20 test results and coverage; stop/start and verify that the 18 rows persist. |

[Back to README](../README.md)
