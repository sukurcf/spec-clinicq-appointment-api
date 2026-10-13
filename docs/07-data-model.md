# Data model

## Source contract

`orders.csv` is UTF-8, has one header row, and contains at most 500 data rows. The exact header order is `order_id,customer_id,order_date,amount`.

| Column | Source rule | Target type |
|---|---|---|
| `order_id` | Trimmed, non-empty, case-sensitive stable key | `text` primary key |
| `customer_id` | Trimmed, non-empty fictional identifier | `text`, not null |
| `order_date` | Real date formatted `YYYY-MM-DD` | `date`, not null |
| `amount` | Finite, non-negative decimal; at most two fractional digits; fits `numeric(12,2)` | `numeric(12,2)`, not null, check `>= 0` |

## Target grain

The `orders` table has one row per distinct `order_id`. Its columns are `order_id`, `customer_id`, `order_date`, and `amount`; `order_id` is the primary key. A repeated key from another run updates that row only when one of its other values changed. No separate customer table is needed.

## Quarantine contract

The local quarantine CSV contains the four original source columns plus `reason`. Preserve the rejected source values. Allowed reason codes are `missing_order_id`, `missing_customer_id`, `invalid_order_date`, `invalid_amount`, and `duplicate_order_id`. For a row with several field errors, store only the first code in the field order shown above. A duplicate reason is assigned only after field validation succeeds.

## Deterministic fixture

| Source rows | Fictional values | Result |
|---|---|---|
| 1–18 | `O-1001` through `O-1018`; `C-2001` through `C-2018`; date `2026-10-01`; amount `10.00` | 18 accepted |
| 19 | `O-1019,C-2019,2026-02-30,10.00` | Rejected: `invalid_order_date` |
| 20 | `O-1020,C-2020,2026-10-13,-1.00` | Rejected: `invalid_amount` |

With an empty target, the result is 18 target rows. On an unchanged replay, it remains 18 rows. The source count excludes the header.

## Count definitions

| Count | Meaning |
|---|---|
| `source` | All data rows read, including invalid and duplicate rows |
| `accepted` | Valid first occurrences after within-file duplicate handling |
| `rejected` | Invalid rows plus later valid duplicate rows |
| `inserted` | Accepted keys absent from the target before the transaction |
| `already_present` | Accepted keys present before the transaction, including changed payloads updated by upsert |

For a successful file, `source = accepted + rejected` and `accepted = inserted + already_present`.

[Back to README](../README.md)
