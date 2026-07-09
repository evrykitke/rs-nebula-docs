# Setup

## Prerequisites

- **Rust** (edition 2024 toolchain)
- **PostgreSQL** installed bare metal (not containerized)
- **Docker** for Redis and RabbitMQ
- **Node.js + Angular CLI** for the frontend (`D:\Ak\Dev\Ruru\Pylon-frontend`)

## Layout

| Path | Purpose |
|---|---|
| `D:\Ak\Dev\Ruru\Pylon` | rs-nebula framework workspace (repo: https://github.com/evrykitke/rs-nebula) |
| `D:\Ak\Dev\Ruru\Bin` | All cargo build artifacts (set via `.cargo/config.toml`) |
| `D:\Ak\Dev\Ruru\Pylon-frontend` | Angular frontend (NSwag proxies, RxJS) |
| `D:\Ak\Dev\Ruru\PylonDocs` | This documentation |

## First run

```sh
# start infrastructure (Redis + RabbitMQ)
docker compose up -d

# build and test
cargo build
cargo test

# run the server (binary lands in D:\Ak\Dev\Ruru\Bin\debug)
cargo run -p nebula-server
```

Then open:

- http://127.0.0.1:5000/health — liveness check
- http://127.0.0.1:5000/swagger-ui — interactive API docs
- http://127.0.0.1:5000/api-docs/openapi.json — OpenAPI document (input for NSwag)

## Configuration

Configuration is layered; later layers win:

1. Built-in defaults
2. `{env}.yaml` — `dev.yaml`, `test.yaml` or `prod.yaml`, selected by `NEBULA_ENV` (default `dev`)
3. `{env}.local.yaml` — gitignored overlay for machine-local secrets (e.g. `dev.local.yaml` holds the database URL)
4. `NEBULA__*` environment variables, `__` separates sections: `NEBULA__SERVER__PORT=8080`

Secrets (database/redis/rabbitmq URLs with credentials) go in the local
overlay or environment variables, never in checked-in files. They are
wrapped in a `Secret` type that prints `***` in logs and debug output.

Business reference data is configuration too — the currency table lives
under `currencies:` in the environment file, not in code.

## Running database tests

Integration tests that need Postgres read `NEBULA_TEST_DATABASE_URL`
(pointing at the `nebula_test` database) and skip when it is unset.
