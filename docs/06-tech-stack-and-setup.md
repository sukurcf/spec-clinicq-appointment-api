# Tech stack and setup

## Reference versions

Use these verified version families and lock Python dependencies. Keep the PostgreSQL image pinned; never use `latest`.

| Tool | Version | Use |
|---|---|---|
| Python | 3.12.x | CLI and tests |
| Python standard library | Python 3.12 | `csv`, `argparse`, `json`, `decimal`, `hashlib` |
| psycopg | 3.3.x | PostgreSQL driver; only external runtime dependency |
| PostgreSQL | 18.x; Compose image `postgres:18.6` | One local database |
| Docker Engine | 29.x | Run PostgreSQL locally |
| Docker Compose | v5.x (5.5) | Local database lifecycle |
| uv | 0.12.x | Locked package and environment management |
| pytest | 9.1.x | Required tests |
| pytest-cov / coverage | 7.1.x / 7.16.x | Line and branch coverage |
| Ruff | 0.16.x | Python lint and format checks |
| Git | Installed system version | Source control |

Use `argparse` instead of adding a CLI framework. Do not add another runtime service or dependency.

## Host setup

- **Windows:** use Windows 11 with WSL2 Ubuntu; allocate 4 GiB WSL memory and 4 GiB swap. Install Python 3.12, uv, Git, and Docker Desktop with its WSL2 backend.
- **macOS:** install Python 3.12, uv, Git, and Docker Desktop.
- **Linux:** install Python 3.12, uv, Git, Docker Engine, and Docker Compose.

The learner implementation repository supplies its Compose configuration, `.env.example`, lockfile, and fictional fixture. This specification repository contains Markdown only.

## Database and local configuration

Compose defines one service named `db`, using the pinned PostgreSQL image, database/user `loadlite`, internal port `5432`, and host binding `127.0.0.1:25432`. Do not publish any other host port. The service uses a named volume `loadlite_pgdata` and a `pg_isready` health check. The CLI runs directly on the host.

Copy `.env.example` to the untracked `.env.local`; set a random local-only database credential and the matching `LOADLITE_DATABASE_URL` there. Never commit `.env.local`. The fixed host port is reserved for this project.

## Exact local commands

Run once after cloning to download the locked package and pinned image:

```bash
cp .env.example .env.local
uv sync --locked
docker compose --env-file .env.local pull db
```

After editing `.env.local`, start PostgreSQL and wait for its health check:

```bash
docker compose --env-file .env.local up -d --pull never db
docker compose --env-file .env.local ps
docker compose --env-file .env.local exec -T db pg_isready -U loadlite -d loadlite
```

The database must report `healthy`, and `pg_isready` must exit successfully before a load. Run the offline demo:

```bash
uv run --env-file .env.local --offline loadlite run fixtures/orders.csv --format json --quarantine .local/quarantine/orders-rejected.csv
```

Stop and restart without deleting data:

```bash
docker compose --env-file .env.local stop db
docker compose --env-file .env.local up -d --pull never db
```

The `loadlite_pgdata` named volume survives both commands. Do not use a volume-removing command for normal stop/start. The only data reset is `uv run --env-file .env.local --offline loadlite reset --confirm-delete-local-data`; its confirmation and scope are defined in [08 CLI specification](08-api-specification.md).

## Fictional seed and offline runtime

The only seed is `fixtures/orders.csv`: 20 deterministic fictional records, with 18 valid rows and 2 invalid rows as defined in [07 Data model](07-data-model.md). A fresh target starts empty; there is no generated customer or order data. The seed file is never modified by the loader.

Initial package and image downloads need a connection. After `uv sync --locked` and `docker compose ... pull db` have completed, use `uv run --offline` and Compose `--pull never`. Runtime, acceptance tests, and fixture processing use local files and PostgreSQL only; they work with external network access disabled.

## Health and errors

The Compose health check runs `pg_isready -U loadlite -d loadlite`. For a missing input, invalid file structure, more than 500 rows, or unavailable PostgreSQL, the CLI exits nonzero with the concise messages in [08 CLI specification](08-api-specification.md), writes no success JSON, and does not leave partial database writes.

[Back to README](../README.md)
