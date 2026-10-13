# Development and CI quality

## Git workflow

1. Create a short branch from `main` for one requirement or test improvement.
2. Make a focused change and run Ruff plus the relevant pytest tests.
3. Open a pull request with linked requirement IDs and exact test evidence.
4. Merge only after required CI checks pass.

Use clear commit messages such as `feat: add order validation` and `test: verify changed-key upsert`. Never commit `.env.local`, generated quarantine output, real data, or credentials.

## CI job

Run CI on **Python 3.12.x only**; do not configure a multi-version matrix.

| Job | Steps | Pass condition |
|---|---|---|
| Quality and PostgreSQL tests | Set up Python 3.12.x and locked dependencies; run Ruff checks; start PostgreSQL 18; run all 20 pytest cases and coverage. | Lint passes, all tests pass, `loadlite` coverage is at least 80% line and 70% branch. |

GitHub Actions may download locked tools for CI. The local demo and acceptance tests work offline after the first package and image downloads.

## Local environment

| Variable | Purpose | Handling |
|---|---|---|
| `LOADLITE_DATABASE_URL` | Connect the host CLI to local PostgreSQL. | Store only in untracked `.env.local`; host is `127.0.0.1`, port `25432`. |
| `LOADLITE_QUARANTINE_PATH` | Optional default quarantine location. | Defaults to `.local/quarantine/orders-rejected.csv`; keep generated output untracked. |

Use a local-only credential. Do not print the connection value in logs or test output. The reference Compose port is fixed and must bind to loopback only.

## Definition of Done

- One CSV contract and the 500-row limit are enforced.
- The 20-row fixture yields 18 accepted rows, two reason-coded rejects, and exactly 18 target rows.
- Same-file replay inserts zero rows; a changed existing key updates one row.
- Database writes roll back as a unit after a failure.
- All 20 catalog cases pass on Python 3.12.x and coverage thresholds are met.
- Compose health, loopback binding, offline run, persisted stop/start, clear errors, and confirmed reset match the spec.
- Documentation and sample values are fictional; no credential or generated implementation file is committed to this specification repository.

[Back to README](../README.md)
