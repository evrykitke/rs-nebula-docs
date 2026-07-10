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
- Framework entities (tenants, users, roles, refresh tokens, permission
  grants, the login directory) key on application-generated UUIDv4
  primary keys — ids leak no ordinal information and stay unique across
  tenant databases. The exception is the audit trail, which keeps a
  bigserial id: its rows are append-only log lines, not domain entities.
- `/health/ready` reports database state and returns 503 while it is down.

## Money

`nebula::Money` = exact `Decimal` amount + ISO 4217 `Currency` (with minor
units: KES/USD 2, JPY 0, BHD 3). Currencies are not hardcoded: they live
in the `currencies` table, pre-populated by a framework migration with
the world's currencies as **system rows that cannot be deleted**.
`CurrencyModule` exposes them — `GET /currencies` is anonymous (the
onboarding form needs it before any account exists), while adding a
deployment-specific unit (`POST`) and deleting one (`DELETE`) require
the tenant-settings permission. The kernel builds a `CurrencyRegistry`
from the table at boot (`ModuleContext::currencies()`); entries under
`currencies:` in the environment yaml are folded in on top for
app-specific units. Cross-currency arithmetic is an error — only
`checked_add`/`checked_sub` exist. Rounding is explicit banker's
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
  its admin account in one step, optionally with the currency the
  company operates in (validated against the currency table).
- **Company profile** (`/auth/tenant/profile`): display name, tax
  registration identifiers (tax PIN, VAT number) and the default
  currency. Readable by any user of the tenant; editing requires the
  tenant-settings permission. `POST /auth/tenant/logo` uploads the
  company logo (png/jpg/svg/webp, ≤ 1 MiB) to
  `{files.root}/{tenant-id}/`, served back at `/public/{tenant-id}/…` —
  the `files.root` directory (default `public/`) is mounted read-only at
  `/public` by the web layer.
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
- **Sessions**: login returns an access/refresh token pair. Access JWTs
  embed the user's security stamp; password changes, deactivation and
  2FA changes rotate the stamp, killing every outstanding token.
  Refresh tokens are stored only as SHA-256 hashes and **rotate on every
  use** — reusing a consumed token is treated as theft and revokes all
  of that user's sessions. `POST /auth/token/refresh` exchanges the
  pair, `POST /auth/logout` revokes one. Failed logins trip a temporary
  lockout (423), and wrong login vs wrong password are indistinguishable
  (401).
- **Onboarding**: one `register` call creates the company and its admin.
  Admin rights are transferable later — admins create and list team
  members (`/auth/users`), grant or revoke admin
  (`PUT /auth/users/{id}/admin`, never on themselves so a tenant cannot
  lose its last admin), and users change passwords at `/auth/password`.
- Handlers take the `CurrentUser` extractor to require authentication.

## Authorization

Roles and permissions in the ASP.NET Zero style, on top of the user
entity above.

- **Permissions are defined in code**: modules contribute trees of
  hierarchical dot-named `PermissionDef`s
  (`Pages.Administration.Users.Edit`, ...) via
  `ModuleContext::add_permissions`. The kernel validates all trees into
  a single registry at boot — duplicate or malformed names fail fast.
  Granting a parent never implicitly grants its children; every grant
  is explicit.
- **Roles carry grants in the database**. Each tenant has its own
  roles. The static `Admin` role is seeded at registration, assigned to
  the registering user, implicitly holds every permission and cannot be
  deleted — so a fresh company is fully able to administer itself.
- **Per-user overrides**: a user can be granted a permission no role
  gives them, or denied one their roles grant. Resolution order: the
  user override decides first (deny beats grant), then static-role
  membership, then role grants.
- **Handlers check with the `Authz` extractor**:
  `authz.require(names::USERS_EDIT).await?` answers 403 when the
  permission is missing and 500 when the name was never defined (a typo
  must never silently pass or deny).
- **Management endpoints**: role CRUD with grants (`/auth/roles`), user
  role assignment (`PUT /auth/users/{id}/roles`), per-user overrides
  (`/auth/users/{id}/permissions`), the definition tree
  (`GET /auth/permissions`) and the caller's effective set
  (`GET /auth/me/permissions`). Users cannot edit their own roles or
  overrides, and admin rights (`PUT /auth/users/{id}/admin`) are never
  self-revocable — a tenant cannot lose its last admin by accident.

## Audit logging

Who did what, from where, and what really changed — one `audit_logs`
table carries three kinds of rows, each stamped with the full request
context: user (from the bearer token), tenant, ip address (proxy header
or socket), user agent, and the `x-request-id` linking the row to traces.

- **Request rows**: middleware records every mutating HTTP request
  (POST/PUT/PATCH/DELETE) with status code and duration. Bodies are
  never read or stored — they can carry passwords. Reads opt in via
  `audit.include_reads`.
- **Entity rows**: handlers record `create`/`update`/`delete` with
  before/after snapshots stored as jsonb, using the `Audit` extractor
  (`audit.0.updated("user", id, &before, &after).await`). Snapshot the
  client-safe view of an entity, never the raw row. The built-in auth
  module snapshots all its admin mutations (users, roles, overrides,
  tenant policy).
- **Event rows**: plain human-readable lines without snapshots —
  "boss logged in", "failed login attempt for x", password changes,
  2FA toggles — via `Recorder::event`.

Reading the trail (`AuditModule`, guarded by
`Pages.Administration.AuditLogs.View`, tenant-scoped):

- `GET /audit/logs` — newest first, paged, filterable by `action`,
  `entity_type`, `user_id`
- `GET /audit/logs/{id}` — one row with full snapshots
- `GET /audit/logs/{id}/diff` — the **what-changed view**: a field-level
  diff listing only the fields that differ between the snapshots

Retention: the trail grows fast, so a background job prunes rows past
their window (every `audit.prune_interval_secs`). The system default is
`audit.retention_days` (30); each tenant can override it at
`PUT /audit/retention` up to `audit.retention_max_days` (180 — six
months). Audit writes are failure-contained: a broken audit store logs
an error but never fails the mutation it describes.

## Background jobs

Long-running work leaves the request path through [apalis] workers on
Redis-backed queues. Enable with `jobs.enabled` (workers connect through
`redis.url`; boot fails fast when Redis is unreachable), and the kernel
runs the worker monitor alongside the web host with graceful shutdown.

- **Enqueue from handlers** through the `Jobs` client in request
  extensions: `jobs.enqueue("emails", SendEmail { .. }).await?` — jobs
  are serialized to Redis and survive restarts; a crashed worker's job
  is re-queued after a visibility timeout.
- **Contribute workers from modules** in `configure`:

  ```rust
  let jobs = ctx.jobs().expect("requires jobs.enabled");
  let storage = jobs.storage::<SendEmail>("emails");
  ctx.add_worker(move |monitor| {
      monitor.register(
          WorkerBuilder::new("emails")
              .backend(storage)
              .build_fn(send_email),
      )
  });
  ```

Built-in workers:

- **Audit retention pruner** — the pruning pass runs as a cron worker on
  `audit.prune_interval_secs` (when jobs are disabled it falls back to a
  plain in-process interval, so retention holds either way).
- **Tenant migrations** — `MigrateTenants` rolls framework and
  application migrations across tenant databases *without a restart*,
  so existing tenants pick up newly deployed features immediately.
  `POST /auth/tenant/migrate` (tenant-settings permission) queues it for
  the caller's tenant; the queue action is recorded in the audit trail.

[apalis]: https://crates.io/crates/apalis

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

Audit logs with entity snapshots, Apalis jobs, RabbitMQ events, Redis
caching.
