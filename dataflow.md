# Dataflow

## Request lifecycle

```
client
  │
  ▼
SetRequestId ── generates x-request-id (UUID)
  │
  ▼
TraceLayer ──── opens a tracing span for the request
  │
  ▼
CatchPanic ──── converts handler panics into problem+json 500s
  │
  ▼
Timeout ─────── aborts requests exceeding server.request_timeout_secs (408)
  │
  ▼
Router ──────── module routes | /health | /swagger-ui | fallback (404)
  │
  ▼
Handler ─────── returns nebula::Result<T>
  │
  ├─ Ok(T)          → serialized response
  └─ Err(Error)     → RFC 9457 problem+json (5xx details logged, not sent)
  │
  ▼
PropagateRequestId ── echoes x-request-id on the response
```

## Boot sequence

1. `Kernel::builder().build()`
   - Load configuration (defaults → `{env}.yaml` → `{env}.local.yaml` → `NEBULA__*` env)
   - Validate configuration — invalid config aborts boot with a clear message
   - Initialize `tracing` (level/format from config; `RUST_LOG` overrides)
2. `kernel.init()`
   - Connect the database pool when `database.url` is set, and ping it —
     an unreachable database fails the boot, not the first request
   - Apply registered migrations when `database.auto_migrate` is on
   - Each module's `configure()` runs in registration order, contributing
     routes and receiving the pool via `ModuleContext::db()`
   - Framework routes and resilience layers wrap the composed router
3. `app.serve()`
   - Bind and serve; ctrl-c triggers graceful shutdown (in-flight requests drain)

## Configuration flow

```
defaults  <  {env}.yaml  <  {env}.local.yaml (gitignored)  <  NEBULA__* env vars
```

`{env}` is `dev`, `test` or `prod`, selected by `NEBULA_ENV`. Secrets
travel only through the local overlay or the environment and are held in
`Secret`, which redacts itself in Debug/Display output.

## Tenant resolution (when multitenancy is enabled)

```
request
  │
  ├─ no tenant header ──────────── host context: main database
  │
  └─ X-Tenant: <name>
       │
       ▼
     directory lookup (main db)
       ├─ unknown ───────────────── 404 problem+json
       ├─ inactive ──────────────── 403 problem+json
       └─ active
            ├─ has connection_string → tenant's own pool (lazy, cached)
            └─ none ─────────────────→ shared main database
       │
       ▼
     handler extracts CurrentTenant (identity) and TenantDb (connection)
```

## Login flow

```
POST /auth/login {login, password}
  │  (tenant context from the X-Tenant header, as everywhere)
  ├─ bad credentials ──────────── 401 (enumeration-safe)
  ├─ repeated failures ────────── 423 locked (temporary)
  ├─ user has an authenticator ── {status: two_factor_required, two_factor_token}
  │      └─ POST /auth/login/two-factor {code}   code = TOTP or recovery code
  │            └─ {status: success, access_token}
  ├─ company mandates 2FA, none set up
  │      └─ {status: two_factor_setup_required, two_factor_token}
  │            └─ POST /auth/two-factor/setup → secret + otpauth:// URL
  │            └─ POST /auth/two-factor/confirm {code} → recovery codes
  │            └─ log in again
  └─ otherwise ────────────────── {status: success, access_token}
```

## Future flows (as subsystems land)

- **Events**: domain events published in-process; integration events via RabbitMQ.
- **Jobs**: long-running work handed to Apalis workers, never done in request handlers.
