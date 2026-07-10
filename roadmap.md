# Roadmap

Foundation (done, 2026-07):

- [x] Cargo workspace: `nebula` (framework), `nebula-server` (host), `nebula-tests`
- [x] docker-compose for optional local Redis + RabbitMQ
- [x] Layered configuration with validation and secret redaction
- [x] Tracing-based logging
- [x] Kernel + module system, graceful shutdown
- [x] Web defaults: health, OpenAPI/Swagger, timeout, panic containment, request ids, problem+json errors
- [x] Clock abstraction (UTC-only core) and decimal money groundwork
- [x] Proof-of-concept test suite

Database layer (done, 2026-07):

- [x] SeaORM connectivity: config-driven pool, boot-time ping, `/health/ready`
- [x] Migrations via `KernelBuilder::with_migrations`, `auto_migrate` toggle
- [x] `Repository<E>` common verbs + `db::transaction` unit of work
- [x] `Money` (Decimal + ISO currency): checked arithmetic, banker's rounding, lossless allocation
- [x] Config refactor: env-named yaml files (dev/test/prod + gitignored .local overlay), currencies configured not hardcoded

Multitenancy (done, 2026-07):

- [x] Tenant directory in the main database (framework-owned migrations)
- [x] `TenantManager`: directory CRUD, lazy per-tenant connection pools
- [x] Header-based resolution middleware (`multitenancy.header`, default `X-Tenant`): unknown → 404, inactive → 403
- [x] `CurrentTenant` / `TenantDb` extractors; tenants without their own connection string share the main database
- [x] Boot migrates directory, main database and every active tenant database (`auto_migrate`)
- [x] Toggleable via `multitenancy.enabled` for self-hosted single-database deployments

Authentication (done, 2026-07):

- [x] Exhaustive user entity (identity, Argon2id credentials, lockout, 2FA state, preferences, soft delete), per-tenant unique
- [x] TOTP two-factor for authenticator apps + hashed single-use recovery codes
- [x] Company registration: tenant + admin account from the registration email/password
- [x] Company-mandated 2FA (`tenants.require_two_factor`) and per-user opt-in
- [x] JWT sessions with security-stamp invalidation; login/two-factor/profile endpoints
- [x] Refresh tokens: hashed at rest, rotation on use, reuse detection revoking all sessions, logout
- [x] Team onboarding: admin creates/lists users, transferable admin rights (self-demotion blocked), change password

Authorization (done, 2026-07):

- [x] Hierarchical permission definitions in the ASP.NET Zero style
      (`Pages.Administration.Users.Edit`, ...) contributed by modules,
      validated into a registry at boot
- [x] Roles with permission grants; static `Admin` role (implicitly all
      permissions, undeletable) seeded and assigned at registration
- [x] Per-user overrides: explicit grant without a role, explicit deny
      that beats every role grant
- [x] `Authz` extractor — `authz.require(names::USERS_EDIT).await?`
- [x] Management endpoints: role CRUD, user role assignment, per-user
      overrides, permission tree, caller's effective permissions
- [x] Admin endpoints guarded by permissions (interim admin flag retired
      from guards); nobody can edit their own roles or overrides

Audit logging (done, 2026-07):

- [x] `audit_logs` with full request context: user, tenant, ip address,
      user agent, request id
- [x] Request rows for every mutating HTTP request (status, duration;
      bodies never recorded), reads opt-in
- [x] Entity rows with before/after jsonb snapshots via the `Audit`
      extractor; auth admin mutations snapshotted out of the box
- [x] Event rows — plain human-readable lines (logins, failed attempts,
      password changes, 2FA toggles) without snapshots
- [x] Trail endpoints with a field-level what-changed diff view,
      permission-guarded and tenant-scoped
- [x] Retention pruning job: 30-day system default, per-tenant override
      capped at six months
- [x] Configuration moved to a `config/` folder

Background jobs (done, 2026-07):

- [x] Apalis workers on Redis-backed queues (`jobs.enabled`), monitor
      run by the kernel alongside the web host
- [x] `Jobs` client in request extensions for enqueueing; modules
      contribute workers via `ModuleContext::add_worker`
- [x] Audit retention pruner as a cron worker (in-process interval
      fallback when jobs are disabled)
- [x] Tenant migration job: existing tenants pick up new features
      without a restart; `POST /auth/tenant/migrate`, audited

Next milestones, in intended order:

1. **Events** — in-process domain events; RabbitMQ integration events.
2. **File storage** — tenant uploads under `public/{tenant-id}/`.
3. **Tenant database provisioning** — generated tenant databases named
   `{slug}-{key}` (e.g. `acme-5jy78k`), migrated by the tenant
   migration job at creation.
4. **Caching** — Redis-backed cache abstraction.
5. **Frontend pipeline** — Angular frontend with NSwag service proxies
   generated from `/api-docs/openapi.json`, RxJS interactivity.
6. **Tooling** — scaffolding CLI for repetitive tasks (after the above are stable).
