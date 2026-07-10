# Frontend & client generation

The framework treats the OpenAPI document as the contract between the
backend and any client. Every module contributes its endpoints to one
document, and typed clients are generated from it — so adding a module
(accounting, sales, …) never means hand-writing HTTP calls.

## How the document is assembled

Each module describes its handlers with `utoipa` annotations and hands
the resulting document to the kernel during `configure`:

```rust
impl Module for AuditModule {
    fn configure(&self, ctx: &mut ModuleContext) {
        ctx.add_api(nebula::module::build_openapi(|| {
            <ApiDoc as utoipa::OpenApi>::openapi()
        }));
        // routes, permissions, workers ...
    }
}
```

The kernel merges every contribution into the base document and serves
it at `/api-docs/openapi.json` (browsable at `/swagger-ui`).
`build_openapi` evaluates the derive on a dedicated big-stack thread —
the expansion for a module with many endpoints is a single deeply
nested expression that can overflow the default stack in unoptimized
builds. Self-referential schema types need `#[schema(no_recursion)]`
on the recursive field.

## Generating service proxies

The Angular client ([ng-nebula](https://github.com/evrykitke/ng-nebula))
generates its service proxies with [NSwag](https://github.com/RicoSuter/NSwag):

```bash
# with a nebula server running (NEBULA_API_URL overrides localhost:5000)
npm run generate-proxies
```

The script pulls `/api-docs/openapi.json` into `openapi.json`, runs
NSwag with the Angular template (Luxon dates, one `…ServiceProxy` class
per tag, an `API_BASE_URL` injection token), and applies post-generation
fixes. Both the document and the generated `service-proxies.ts` are
committed, so API changes show up in code review and builds do not need
a live server.

When a new backend module lands, regenerating the clients is that same
single command.

## CORS

Browser frontends run on a different origin than the API. The origins
allowed to call the API are configured per environment:

```yaml
server:
  cors_origins:
    - http://localhost:4200   # the Angular dev server
```

An empty list (the default) disables CORS entirely.

## What the client does with it

- **Onboarding** — `/register` creates a company and its admin account
  in one form (the workspace identifier is derived from the company
  name), then signs the new admin straight in.
- **Authentication** — users sign in with credentials alone: the server
  resolves the workspace through the login directory and every login
  response names it, which the client adopts for the tenant header. If
  the same credentials exist in several companies the user picks the
  workspace from the `tenant_selection` answer. Permissions come from
  `GET /auth/me/permissions` (they are not in the JWT). Refresh tokens
  rotate; a 401 triggers one transparent refresh-and-retry. Both
  two-factor branches are handled: code entry, and company-mandated
  enrollment during sign-in.
- **Tables** — entity lists declare a `TableConfig` (fluent
  `ColumnBuilder` columns, row actions, feature toggles) rendered by a
  reusable `<app-data-table>` driven by a `TableDataSource` that maps a
  table query onto the module's list endpoint.
- **Administration pages** — users, per-user roles and grant/deny
  overrides, roles with the permission tree from `/auth/permissions`,
  the audit trail (rows expand inline to the request context and change
  set; a full timeline page per entry), and tenant settings.
