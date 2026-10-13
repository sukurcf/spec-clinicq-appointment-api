# CLI specification

The single required interface is the `loadlite` command. The host process connects using `LOADLITE_DATABASE_URL` from `.env.local`; Compose keeps PostgreSQL on `127.0.0.1:25432`.

## Run a batch

```text
loadlite run <orders.csv> --format json [--quarantine <path>]
```

- `<orders.csv>` is required. Use `fixtures/orders.csv` for the demo.
- `--format json` is required and writes one summary object to stdout.
- If omitted, `--quarantine` defaults to `.local/quarantine/orders-rejected.csv`.
- The quarantine file has source columns plus `reason`. Row-level rejects do not fail the batch command.
- Successful JSON has exactly these five integer fields and no extra summary keys:

```json
{"source":20,"accepted":18,"rejected":2,"inserted":18,"already_present":0}
```

## Validation and key behavior

The header must be exactly `order_id,customer_id,order_date,amount`, with at most 500 data rows. Validate the required text fields, a real `YYYY-MM-DD` date, and a finite non-negative decimal amount with at most two fractional digits. Reject whole-file structural errors before database writes.

For each row, check fields before duplicate keys. Retain the first valid source occurrence of an `order_id`; quarantine every later valid occurrence as `duplicate_order_id`. Invalid rows do not reserve a key. Use bound parameters for database values.

All accepted rows are applied in one transaction. A new key is inserted. For a key already in the target, unchanged values are a no-op; changed `customer_id`, `order_date`, or `amount` update that row. Such a row counts as `already_present`, not `inserted`. Replaying the same input checksum must add no rows. A database error rolls back all inserts and updates and returns no success summary.

## Count contract

`source` excludes the header. `accepted` counts valid unique source rows, whether inserted or already present. `rejected` counts invalid rows and later valid source duplicates. `inserted` counts new target keys. `already_present` counts keys found in the target before the batch, including changed-payload upserts.

Expected fixture outputs:

| Run | `source` | `accepted` | `rejected` | `inserted` | `already_present` | Target rows |
|---|---:|---:|---:|---:|---:|---:|
| First run against empty database | 20 | 18 | 2 | 18 | 0 | 18 |
| Same-file rerun | 20 | 18 | 2 | 0 | 18 | 18 |
| One changed payload for an existing key | 1 | 1 | 0 | 0 | 1 | Still 1 |

## Reset local data

```text
loadlite reset [--confirm-delete-local-data]
```

Without the flag, exit nonzero with `loadlite: explicit confirmation required; retry with --confirm-delete-local-data` and make no changes. With the flag, clear only the LoadLite `orders` target and its generated quarantine output. Do not remove the source fixture or unrelated local data.

## Fatal errors

Fatal errors go to stderr, have no traceback in normal CLI output, return nonzero, and do not print a success JSON object.

| Condition | Required message |
|---|---|
| Missing input | `loadlite: input file not found` |
| Wrong or malformed header | `loadlite: expected header order_id,customer_id,order_date,amount` |
| More than 500 rows | `loadlite: maximum 500 data rows` |
| Database unavailable | `loadlite: PostgreSQL unavailable at 127.0.0.1:25432` |

[Back to README](../README.md)
