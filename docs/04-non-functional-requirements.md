# Non-functional requirements

Targets below are acceptance criteria, not measured results.

| ID | Category | Requirement | Verification |
|---|---|---|---|
| NFR-LOAD-01 | Performance | With PostgreSQL healthy, validate and load 500 valid rows in at most 10 seconds on the reference laptop; exclude initial downloads and database startup. | Timed local acceptance test |
| NFR-LOAD-02 | Reliability | A single-file load is all-or-nothing in PostgreSQL. A failed write leaves the target unchanged. | Inject a database failure after a write and verify rollback |
| NFR-LOAD-03 | Idempotency | Replaying identical CSV bytes creates no rows; a changed payload for an existing key updates exactly one row. | Repeat and upsert integration tests |
| NFR-LOAD-04 | Data quality | Every invalid data row is preserved in the quarantine output with one deterministic reason code. | Fixture and field-validation tests |
| NFR-LOAD-05 | Portability | Runtime and tests use only cached packages/images, fictional fixtures, and the local database after initial downloads. | Run the demo with external access disabled |
| NFR-LOAD-06 | Security | Publish PostgreSQL only on `127.0.0.1:25432`; keep local credentials out of Git; use fictional identifiers only. | Resolved Compose configuration and repository review |
| NFR-LOAD-07 | Maintainability | `loadlite` has at least 80% line and 70% branch coverage. | pytest-cov report and CI gate |
| NFR-LOAD-08 | Clarity | Missing input, invalid file structure, over-limit input, and unavailable database produce concise actionable errors without tracebacks. | Negative CLI tests |
| NFR-LOAD-09 | Durability | Normal stop/start retains the PostgreSQL named volume; only the explicitly confirmed reset clears LoadLite data. | Persisted restart and reset tests |

## Reference hardware

| Item | Proposed minimum |
|---|---|
| Host | 8 GiB RAM, 4 CPU cores, 20 GiB free disk |
| WSL2, if used | 4 GiB memory and 4 GiB swap |
| Containers | One PostgreSQL 18 service, with a proposed 1 GiB memory budget |
| Dataset | Up to 500 rows; the required fixture has 20 |

The trainer can adjust resource budgets for a specific machine. Do not report proposed targets as measured performance.

[Back to README](../README.md)
