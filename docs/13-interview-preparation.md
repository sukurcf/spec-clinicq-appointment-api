# Interview preparation

## Ten LoadLite questions

1. **How do the five counts relate?** `source = accepted + rejected`; `accepted = inserted + already_present`. The header is not a source row.
2. **Why is `order_id` a stable key?** It identifies one order across file replays and prevents duplicate target rows.
3. **When does a source duplicate get rejected?** After field validation; the first valid row wins, and later valid rows with the same case-sensitive key are quarantined.
4. **How do you validate `order_date` and `amount`?** Require a real `YYYY-MM-DD` date and a finite non-negative decimal that fits `numeric(12,2)`.
5. **Why quarantine invalid rows instead of dropping them?** The original values and a stable reason remain available for correction and review.
6. **What does the transaction protect?** Every accepted insert or update commits together; a database error rolls back the entire batch.
7. **What happens on a same-file replay?** All valid keys are already present, so `inserted` is zero and the target count remains 18.
8. **How is a changed payload handled?** Upsert updates the existing row for that key; it counts as `already_present`, not as a new insert.
9. **How is local access limited and made repeatable?** Compose binds `127.0.0.1:25432`; a pinned local image and cached packages allow offline runtime after initial downloads.
10. **What do line and branch coverage measure here?** Line coverage counts executed package lines; branch coverage checks both sides of decisions such as valid/invalid and new/existing key.

## One-minute explanation

“LoadLite is a small Python batch loader for one fictional orders CSV. It validates a strict four-column contract, keeps the first valid occurrence of each order key, and quarantines invalid or duplicate records with a reason. Accepted rows are inserted or upserted in one PostgreSQL transaction. The CLI reports exact source, accepted, rejected, inserted, and already-present counts. I verified replay, changed payloads, rollback, local health, offline operation, and persistence with a 20-case test suite.”

[Back to README](../README.md)
