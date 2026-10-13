# Functional requirements

All five requirements are Must scope. See [08 CLI specification](08-api-specification.md) for command details and [09 tests](09-testing-strategy-and-test-cases.md) for traceability.

| ID | Title | Priority |
|---|---|---|
| FR-LOAD-01 | Read the CSV contract | Must |
| FR-LOAD-02 | Validate and quarantine rows | Must |
| FR-LOAD-03 | Deduplicate and load transactionally | Must |
| FR-LOAD-04 | Report exact counts | Must |
| FR-LOAD-05 | Operate locally and safely | Must |

## Must requirements

### FR-LOAD-01 — Read the CSV contract

**Given** a UTF-8 `orders.csv` with the exact header `order_id,customer_id,order_date,amount` and no more than 500 data rows, **when** the CLI reads it, **then** it counts data rows (not the header) and validates the file before any database write. Missing files, malformed CSV, wrong headers, or over-limit files produce a clear nonzero error and no database changes.

### FR-LOAD-02 — Validate and quarantine rows

**Given** a structurally valid file, **when** a row has an invalid required field, **then** the CLI preserves its source values in the quarantine CSV and adds the applicable stable reason code. Valid rows are not rejected because another row is bad.

### FR-LOAD-03 — Deduplicate and load transactionally

**Given** valid rows, **when** the batch is loaded, **then** the first valid occurrence of each source `order_id` is retained and later valid occurrences are quarantined. All accepted inserts and updates use one PostgreSQL transaction. A database failure rolls back the entire batch.

**Given** an accepted `order_id` already exists, **when** its payload is unchanged, **then** the row is left unchanged; **when** its payload differs, **then** the existing order is updated without creating a second row.

### FR-LOAD-04 — Report exact counts

**Given** a completed load, **when** JSON output is requested, **then** stdout contains exactly five integer fields: `source`, `accepted`, `rejected`, `inserted`, and `already_present`. For the empty-target fixture, they are `20`, `18`, `2`, `18`, and `0`. For the same-file rerun, they are `20`, `18`, `2`, `0`, and `18`, and the target remains at 18 rows.

### FR-LOAD-05 — Operate locally and safely

**Given** the documented Compose service, **when** it starts, **then** PostgreSQL becomes healthy on host `127.0.0.1:25432`. Normal stop/start preserves the named volume. **When** reset is requested without explicit confirmation, **then** it fails without deleting data; the confirmed reset affects only LoadLite data.

## Business and data rules

| ID | Rule |
|---|---|
| BR-01 | The header is exactly `order_id,customer_id,order_date,amount`, in that order; there is one source file and at most 500 data rows. |
| BR-02 | Trim surrounding whitespace from text fields. `order_id` and `customer_id` must be non-empty; key comparison is case-sensitive. |
| BR-03 | `order_date` must be a real calendar date in `YYYY-MM-DD`. `amount` must be a finite, non-negative decimal with at most two fractional digits and fit `numeric(12,2)`. |
| BR-04 | Validate a row before checking its key against earlier source rows. The first valid occurrence wins; each later valid occurrence is rejected as `duplicate_order_id`. |
| BR-05 | Invalid rows use one reason code: the first failing field in order `order_id`, `customer_id`, `order_date`, `amount`; duplicate detection follows field validation. |
| BR-06 | `order_id` is the stable target key. Existing keys are upserted; changed payloads update the row and count as `already_present`, not `inserted`. |

Fatal file or database errors write a concise message to stderr, return nonzero, and never print a success summary. Row-level rejects are successful batch outcomes and are counted in the JSON summary.

[Back to README](../README.md)
