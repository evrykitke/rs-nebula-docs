# Architecture

rs-nebula is a Domain-Driven Design application framework inspired by
ASP.NET Boilerplate. Guiding principles:

- **Composition over hardcoding** — applications are assembled from modules.
- **Configuration over convention-breaking** — behavior is configured, not patched.
- **"What could go wrong?"** — every subsystem ships with failure containment by default.
- **Developer as artist, not plumber** — the kernel does the plumbing once.

## Workspace

| Crate | Role |
|---|---|
| `nebula` | The framework library: kernel, module system, config, errors, web plumbing, time/decimal primitives |
| `nebula-server` | The host binary; `main.rs` is a one-liner that runs the kernel |
| `nebula-tests` | Proof-of-concept tests demonstrating and pinning framework behavior |

## Kernel and modules

The kernel (`nebula::kernel`) bootstraps everything:

```rust
Kernel::builder()
    .add_module(MyModule)
    .build()?   // loads config, initializes logging, fails fast if invalid
    .run()      // serves until ctrl-c, then drains in-flight requests
    .await
```

A `Module` contributes functionality through a `ModuleContext` (routes today;
jobs, event handlers and entities as those subsystems land). Modules are
configured in registration order.

## Built-in web defaults

Every application gets, with zero code:

- `/health` liveness endpoint
- OpenAPI 3.1 document + Swagger UI (`utoipa`)
- Request timeout (configurable, default 30 s)
- Panic containment — a panicking handler becomes a 500, never a crash
- `x-request-id` generation and propagation for tracing
- RFC 9457 `application/problem+json` for **all** errors including 404s;
  5xx responses never leak internal details

## Database layer

- `nebula::db` — SeaORM/sqlx pool, connected by the kernel when
  `database.url` is set and verified with a ping at boot. Pool sizing and
  connect/acquire/idle timeouts come from configuration.
- `nebula::Repository<E>` — the common data-access verbs for any SeaORM
  entity (ABP `IRepository` style); a missing row is `Error::NotFound`,
  which surfaces as a 404.
- `nebula::db::transaction` — unit of work: commit on `Ok`, rollback on
  `Err`, so partial writes never survive a failure.
- Migrations: applications register a SeaORM migrator with
  `KernelBuilder::with_migrations::<M>()`; pending migrations run at boot
  when `database.auto_migrate` is on.
- `/health/ready` reports database state and returns 503 while it is down.

## Money

`nebula::Money` = exact `Decimal` amount + ISO 4217 `Currency` (with minor
units: KES/USD 2, JPY 0, BHD 3). Currencies are not hardcoded: the
application declares them under `currencies:` in its environment yaml and
the kernel builds a `CurrencyRegistry` from it at boot
(`ModuleContext::currencies()`). Cross-currency arithmetic is an error —
only `checked_add`/`checked_sub` exist. Rounding is explicit banker's
rounding to minor units; `allocate(n)` splits an amount into parts that
differ by at most one minor unit and always sum back to the whole.

## Multitenancy

Toggleable with `multitenancy.enabled` — off means single-tenant against
the main database (self-hosted mode), with zero tenancy overhead.

When enabled:

- The main database holds the **tenant directory** (`tenants` table,
  created by framework-owned migrations tracked separately in
  `nebula_migrations`).
- Each tenant may carry its own connection string (database-per-tenant)
  or none, in which case it shares the main database.
- A request names its tenant in a header (`multitenancy.header`, default
  `X-Tenant`). The middleware resolves it against the directory: unknown
  tenants get 404, inactive ones 403, both as problem+json. No header
  means host context.
- Handlers use the `CurrentTenant` extractor for identity and `TenantDb`
  for the right connection — the tenant's own pool (created lazily,
  cached) or the main database.
- With `auto_migrate` on, boot migrates the directory, the main database,
  and every active tenant database.
- `TenantManager` (from `ModuleContext::tenants()`) exposes directory
  lookups and tenant creation with name validation and duplicate checks.

## Authentication

`AuthModule` gives every application a complete auth surface. The user
entity is deliberately exhaustive: identity, Argon2id password hash,
confirmation/reset tokens, a security stamp, lockout accounting,
two-factor state, preferences (language, time zone) and soft delete —
unique per tenant, stored in the main database or the tenant's own.

- **Company registration** (`POST /auth/register`): in multitenant mode
  the email and password given at registration create the tenant *and*
  its admin account in one step.
- **Login** (`POST /auth/login`): answers `success` with a JWT access
  token, `two_factor_required`, or `two_factor_setup_required` — the
  latter two with a short-lived bridge token that is useless anywhere
  except the two-factor endpoints.
- **Two-factor** is TOTP (RFC 6238) for authenticator apps: setup returns
  the secret and `otpauth://` URL, confirmation returns ten single-use
  recovery codes (stored hashed). A company can mandate 2FA for all its
  users (`POST /auth/tenant/two-factor`, admins only) — users without an
  authenticator are routed through setup at their next login — and any
  user can opt in or out from their profile (out only if the company
  does not mandate it).
- **Sessions**: JWTs embed the user's security stamp; password changes,
  deactivation and 2FA changes rotate the stamp, killing every
  outstanding token. Failed logins trip a temporary lockout (423), and
  wrong login vs wrong password are indistinguishable (401).
- Handlers take the `CurrentUser` extractor to require authentication.

## Error model

`nebula::Error` is the framework-wide error enum (Validation, Unauthorized,
Forbidden, NotFound, Conflict, Internal, ...). It implements
`IntoResponse`, so handlers simply return `nebula::Result<T>`.

## Precision rules

- **Time**: UTC internally, always. Conversion to a user's time zone happens
  only at the presentation edge (`nebula::time::to_local`). Code takes a
  `Clock` (trait) instead of calling `Utc::now()`, so time-dependent logic is
  testable with `FixedClock`.
- **Money**: `nebula::Decimal` (rust_decimal) — never `f64`. The test suite
  pins the classic `0.1 + 0.2` float trap.

## Planned (see roadmap.md)

ABP-Zero-style permissions, audit logs with entity snapshots, Apalis jobs,
RabbitMQ events, Redis caching.
