# Roadmap

Foundation (done, 2026-07):

- [x] Cargo workspace (`nebula`, `nebula-server`, `nebula-tests`), artifacts → `D:\Ak\Dev\Ruru\Bin`
- [x] docker-compose: Redis + RabbitMQ (Postgres is bare metal)
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

Next milestones, in intended order:

1. **Multitenancy** — directory (main) database, database-per-tenant,
   tenant resolution middleware, toggleable via `multitenancy.enabled`
   for self-hosted single-database deployments.
3. **Authentication & authorization** — users, roles, permissions in the
   ASP.NET Zero style (`Pages.Administration.Users.Edit`, ...), role-based
   with per-user overrides; JWT sessions.
4. **Audit logging** — entity snapshots (before/after) on every mutation.
5. **Background jobs** — Apalis workers for long-running processes.
6. **Events** — in-process domain events; RabbitMQ integration events.
7. **Caching** — Redis-backed cache abstraction.
8. **Frontend pipeline** — Angular app in `Pylon-frontend`, NSwag service
   proxies generated from `/api-docs/openapi.json`, RxJS interactivity.
9. **Tooling** — scaffolding CLI for repetitive tasks (after the above are stable).
