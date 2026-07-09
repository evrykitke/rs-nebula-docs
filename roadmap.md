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

Next milestones, in intended order:

1. **Authentication & authorization** — users, roles, permissions in the
   ASP.NET Zero style (`Pages.Administration.Users.Edit`, ...), role-based
   with per-user overrides; JWT sessions.
2. **Audit logging** — entity snapshots (before/after) on every mutation.
3. **Background jobs** — Apalis workers for long-running processes.
4. **Events** — in-process domain events; RabbitMQ integration events.
5. **Caching** — Redis-backed cache abstraction.
6. **Frontend pipeline** — Angular frontend with NSwag service proxies
   generated from `/api-docs/openapi.json`, RxJS interactivity.
7. **Tooling** — scaffolding CLI for repetitive tasks (after the above are stable).
