# Setup

## Prerequisites

- **Rust** (edition 2024 toolchain) — <https://rustup.rs>
- **PostgreSQL** 14+ — any reachable server: local install, Docker,
  or managed. The framework only needs a connection URL.
- **Docker** (optional) — the provided `docker-compose.yml` starts Redis
  and RabbitMQ for caching and messaging. If you already run these, skip it.

## Getting started

```sh
git clone https://github.com/evrykitke/rs-nebula
cd rs-nebula

# optional: start Redis + RabbitMQ
docker compose up -d

cargo build
cargo test
cargo run -p nebula-server
```

Then open:

- `http://127.0.0.1:5000/health` — liveness check
- `http://127.0.0.1:5000/health/ready` — readiness (includes database state)
- `http://127.0.0.1:5000/swagger-ui` — interactive API docs
- `http://127.0.0.1:5000/api-docs/openapi.json` — OpenAPI document (input for client generators such as NSwag)

## Repository layout

| Path | Purpose |
|---|---|
| `crates/nebula` | The framework library |
| `crates/nebula-server` | Host binary bootstrapped by the kernel |
| `crates/nebula-tests` | Proof-of-concept test suite |
| `dev.yaml` / `test.yaml` / `prod.yaml` | Environment configuration |
| `docker-compose.yml` | Optional Redis + RabbitMQ for local development |

## Configuration

Configuration is layered; later layers win:

1. Built-in defaults
2. `{env}.yaml` — `dev.yaml`, `test.yaml` or `prod.yaml`, selected by `NEBULA_ENV` (default `dev`)
3. `{env}.local.yaml` — gitignored overlay for machine-local secrets
4. `NEBULA__*` environment variables, `__` separates sections: `NEBULA__SERVER__PORT=8080`

Point the framework at your database in `dev.local.yaml` (create it next
to `dev.yaml`; it is gitignored):

```yaml
database:
  url: "postgres://user:password@localhost:5432/nebula"
```

or via the environment: `NEBULA__DATABASE__URL=postgres://...`. Special
characters in credentials must be percent-encoded (`@` → `%40`).

Secrets never go in checked-in files. They are wrapped in a `Secret` type
that prints `***` in logs and debug output.

Business reference data is configuration too — the currency table lives
under `currencies:` in the environment file, not in code.

The auth module additionally needs a signing secret (any long random
string) in the local overlay:

```yaml
auth:
  jwt_secret: "<long random string>"
```

## Running database tests

Integration tests that need PostgreSQL read `NEBULA_TEST_DATABASE_URL`
and skip (with a notice) when it is unset. Point it at a disposable
database — tests create and drop tables, and the multitenancy test
creates an additional database next to it:

```sh
NEBULA_TEST_DATABASE_URL=postgres://user:password@localhost:5432/nebula_test cargo test
```
