# Testing strategy and test cases

Use fictional CSVs and PostgreSQL 18. **Offline acceptance:** after the one-time dependency and image downloads, run the full test suite with external network access disabled. Tests use the local database and fixture only. The complete catalog has 20 distinct Must cases.

## Coverage and test tools

| Tool or measure | Requirement |
|---|---|
| Python | 3.12.x |
| pytest | 9.1.x |
| pytest-cov / coverage | 7.1.x / 7.16.x |
| PostgreSQL | 18.x |
| Coverage scope | Python package `loadlite` |
| Line coverage | At least 80% |
| Branch coverage | At least 70% |

Unit tests cover parsing, validation, counting, and error mapping. PostgreSQL integration tests cover transaction rollback, uniqueness, upsert, persistence, and the Compose connection contract.

## Test catalog

| ID | Input and action | Expected outcome | Priority | FRs |
|---|---|---|---|---|
| TC-LOAD-001 | Empty target; run the 20-row fixture. | Exact JSON `{"source":20,"accepted":18,"rejected":2,"inserted":18,"already_present":0}`; 18 target rows; the two quarantine reasons match the fixture. | Must | FR-LOAD-01, FR-LOAD-02, FR-LOAD-03, FR-LOAD-04 |
| TC-LOAD-002 | Run the same fixture bytes again; verify the source checksum is unchanged. | `{"source":20,"accepted":18,"rejected":2,"inserted":0,"already_present":18}`; target remains 18 rows with unchanged values. | Must | FR-LOAD-03, FR-LOAD-04 |
| TC-LOAD-003 | Target has `O-1001`; load one row with the same key and changed customer, date, and amount. | `{"source":1,"accepted":1,"rejected":0,"inserted":0,"already_present":1}`; one target row remains and contains the changed payload. | Must | FR-LOAD-03, FR-LOAD-04 |
| TC-LOAD-004 | Input has two valid `O-9000` rows, amounts `10.00` then `99.00`. | First payload remains in `orders`; the second row is quarantined as `duplicate_order_id`; counts are `2,1,1,1,0`. | Must | FR-LOAD-02, FR-LOAD-03, FR-LOAD-04 |
| TC-LOAD-005 | One row has date `2026-02-30`. | Calendar validation rejects the date; quarantine reason is `invalid_order_date`; counts are `1,0,1,0,0`. | Must | FR-LOAD-02, FR-LOAD-04 |
| TC-LOAD-006 | One row has amount `-1.00`. | Negative amount maps to `invalid_amount`; no order is persisted; counts are `1,0,1,0,0`. | Must | FR-LOAD-02, FR-LOAD-04 |
| TC-LOAD-007 | One row has amount `ten`. | Non-decimal text is quarantined as `invalid_amount`; the target is untouched; counts are `1,0,1,0,0`. | Must | FR-LOAD-02, FR-LOAD-04 |
| TC-LOAD-008 | One row has blank `customer_id`. | The missing customer identifier is quarantined as `missing_customer_id`; counts are `1,0,1,0,0`. | Must | FR-LOAD-02, FR-LOAD-04 |
| TC-LOAD-009 | One row has blank `order_id`. | The missing stable key is quarantined as `missing_order_id`; counts are `1,0,1,0,0`. | Must | FR-LOAD-02, FR-LOAD-04 |
| TC-LOAD-010 | Header omits `order_date`; target contains one known row. | Nonzero exit with the exact header error; the existing target row remains unchanged and no success JSON is printed. | Must | FR-LOAD-01 |
| TC-LOAD-011 | File contains the correct header and no data rows. | Success with all five counts `0`; no target rows are added. | Must | FR-LOAD-01, FR-LOAD-04 |
| TC-LOAD-012 | Empty target; load exactly 500 unique valid rows. | Success with counts `500,500,0,500,0`; target contains 500 rows. | Must | FR-LOAD-01, FR-LOAD-03, FR-LOAD-04 |
| TC-LOAD-013 | Target has one known row; input contains 501 valid rows. | Nonzero exit with `loadlite: maximum 500 data rows`; the known target row is unchanged. | Must | FR-LOAD-01 |
| TC-LOAD-014 | Test database raises an injected error on the second write in a two-row batch. | Nonzero database error; neither new row remains after rollback; no success JSON. | Must | FR-LOAD-03 |
| TC-LOAD-015 | Run with a nonexistent input path. | Nonzero exit with `loadlite: input file not found`; no database row changes. | Must | FR-LOAD-01 |
| TC-LOAD-016 | Stop PostgreSQL; run a valid one-row file, then restart PostgreSQL and inspect the target. | Nonzero exit with `loadlite: PostgreSQL unavailable at 127.0.0.1:25432`; no traceback or partial row. | Must | FR-LOAD-03, FR-LOAD-05 |
| TC-LOAD-017 | Inspect the resolved Compose port mapping. | The only published service port is `127.0.0.1:25432` to container `5432`; no wildcard bind appears. | Must | FR-LOAD-05 |
| TC-LOAD-018 | Start a fresh Compose database and run the documented health check. | Compose reports `db` healthy and `pg_isready -U loadlite -d loadlite` exits successfully before the load starts. | Must | FR-LOAD-05 |
| TC-LOAD-019 | Load the fixture, stop the database, start it again, and count target rows. | The named volume is retained and the target still has 18 rows. | Must | FR-LOAD-05 |
| TC-LOAD-020 | Load the fixture; run reset without its flag, then with `--confirm-delete-local-data`. | Unconfirmed reset fails and retains 18 rows. Confirmed reset clears only LoadLite target/quarantine data and leaves the source fixture intact. | Must | FR-LOAD-05 |

`TC-LOAD-001` and `TC-LOAD-002` verify that the source total is 20, accepted plus rejected is 20, and the target total is 18 after either run. `TC-LOAD-003` makes the changed-payload rule and count observable.

[Back to README](../README.md)
