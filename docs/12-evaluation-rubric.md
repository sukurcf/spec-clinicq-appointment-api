# Evaluation rubric

Version 1.2 grades the five-day LoadLite outcome. Earlier scope and optional features do not earn marks. The fixed 100 points now emphasize ETL correctness, practical database work, and reproducible local operation.

## Completion gates

All gates are achievable within five days × six hours/day = 30 hours.

- The fixture produces 18 accepted rows, two reason-coded rejects, and 18 target rows.
- Same-file replay inserts zero rows; the changed-payload rule and rollback are demonstrated by tests.
- All 20 required cases pass; `loadlite` coverage reaches 80% line and 70% branch.
- PostgreSQL binds only to loopback; health, offline runtime, persisted restart, and confirmed reset meet the contract.
- The learner explains the row rules, count definitions, and transaction behavior.

## Weighted rubric

| Category | Points | Evidence |
|---|---:|---|
| Functional and data correctness | 40 | Header/row validation, deterministic quarantine, duplicate handling, exact counts, repeat and changed-key behavior. |
| Data/model/SQL/ETL practice | 25 | Clear target grain and stable key, parameterized PostgreSQL writes, one transaction, rollback, and reasonable upsert logic. |
| Tests and CI | 15 | 20 distinct tests, Python 3.12.x CI, and 80% line / 70% branch coverage for `loadlite`. |
| Local reproducibility | 10 | Compose health, exact loopback port, offline run after downloads, persistent stop/start, clear errors, and explicit reset. |
| Documentation and Git | 5 | Concise setup and data contract, focused changes, and readable commit history. |
| Demo and viva | 5 | Five-minute demonstration with accurate explanation of outputs and trade-offs. |
| **Total** | **100** |  |

No unlisted feature changes the score. Missing any completion gate requires rework before a passing grade is recorded.

[Back to README](../README.md)
