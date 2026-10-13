# Glossary and resources

## Glossary

| Term | Meaning |
|---|---|
| Batch | A group of records processed as one load. |
| CSV | Comma-separated text with a header and records. |
| Header | The first CSV row naming the columns. |
| Record | One data row in the source file. |
| Schema | The expected fields, types, and constraints. |
| Validation | Checking a value against a data rule before loading it. |
| Quarantine | A local file of rejected source rows and reasons. |
| Reason code | A stable short label explaining why a row was rejected. |
| Stable key | A source identifier that remains the same across replays. |
| Grain | What one row in a target table represents. |
| Primary key | A column that uniquely identifies a target row. |
| Idempotency | Repeating the same input does not create duplicate results. |
| Checksum | A digest used to identify identical file bytes. |
| Upsert | Insert a new key or update the existing row for that key. |
| Transaction | A database unit whose changes commit together. |
| Commit | Make a successful transaction's changes permanent. |
| Rollback | Discard all uncommitted transaction changes. |
| Decimal | A base-10 numeric representation suited to exact amounts. |
| Bound parameter | A value passed separately from a database statement. |
| Health check | A readiness test for the local PostgreSQL service. |
| Loopback | The host-only network address `127.0.0.1`. |
| Named volume | Docker-managed local storage retained across stop/start. |
| Line coverage | The percentage of measured package lines executed by tests. |
| Branch coverage | The percentage of measured decision outcomes executed by tests. |

## Official documentation

- [Python 3.12 documentation](https://docs.python.org/3.12/)
- [Python `csv`](https://docs.python.org/3.12/library/csv.html)
- [Python `decimal`](https://docs.python.org/3.12/library/decimal.html)
- [Python `argparse`](https://docs.python.org/3.12/library/argparse.html)
- [Python `json`](https://docs.python.org/3.12/library/json.html)
- [psycopg 3 documentation](https://www.psycopg.org/psycopg3/docs/)
- [PostgreSQL 18 documentation](https://www.postgresql.org/docs/18/)
- [PostgreSQL 18 constraints](https://www.postgresql.org/docs/18/ddl-constraints.html)
- [PostgreSQL 18 transactions](https://www.postgresql.org/docs/18/tutorial-transactions.html)
- [Docker Compose documentation](https://docs.docker.com/compose/)
- [uv documentation](https://docs.astral.sh/uv/)
- [pytest documentation](https://docs.pytest.org/en/stable/)
- [pytest-cov documentation](https://pytest-cov.readthedocs.io/)
- [Ruff documentation](https://docs.astral.sh/ruff/)
- [Git documentation](https://git-scm.com/doc)

## Five recommended readings

1. [Reading and writing CSV files](https://docs.python.org/3.12/library/csv.html)
2. [psycopg basic usage](https://www.psycopg.org/psycopg3/docs/basic/usage.html)
3. [PostgreSQL transaction tutorial](https://www.postgresql.org/docs/18/tutorial-transactions.html)
4. [Docker Compose quickstart](https://docs.docker.com/compose/gettingstarted/)
5. [pytest getting started](https://docs.pytest.org/en/stable/getting-started.html)

[Back to README](../README.md)
