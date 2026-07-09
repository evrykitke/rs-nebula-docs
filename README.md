# rs-nebula Documentation

Documentation for [rs-nebula](https://github.com/evrykitke/rs-nebula) — a
Domain-Driven Design application framework for building ERPs in Rust,
inspired by ASP.NET Boilerplate.

## Table of Contents

| Document | Contents |
|---|---|
| [Setup](setup.md) | Prerequisites, project layout, first run, configuration layers, running database tests |
| [Architecture](architecture.md) | Guiding principles, workspace crates, kernel & module system, web defaults, database layer, Money, error model, precision rules |
| [Dataflow](dataflow.md) | Request lifecycle through the middleware stack, boot sequence, configuration flow, future flows |
| [Roadmap](roadmap.md) | Completed milestones and what comes next |

## Quick links

- API playground: `http://127.0.0.1:5000/swagger-ui` (when the server is running)
- OpenAPI document: `http://127.0.0.1:5000/api-docs/openapi.json`
- Health: `http://127.0.0.1:5000/health` · Readiness: `http://127.0.0.1:5000/health/ready`
