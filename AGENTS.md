# Circular

Circular is a Linear-style issue tracker: workspaces, boards, issues with status, priority and assignees, and comments, with live board updates across clients. It is a server-rendered web application written in Rust.

## Stack

| Concern | Choice |
|---|---|
| HTTP | Axum 0.8 on Tokio |
| Database | PostgreSQL 17 via SQLx (raw SQL, compile-time checked queries, `sqlx migrate`) |
| Cache / ephemeral state | Redis 8 via the `redis` crate (Tokio, `ConnectionManager`) |
| Frontend | HTMX + Askama templates, served by the API; no SPA, no JS build step |
| Auth | argon2 password hashing, server-side sessions stored in Redis |
| Errors | `thiserror` + a single `AppError` implementing `IntoResponse` |
| Observability | `tracing`, `tower-http` trace layer |
| Local infra | Docker Compose (`docker compose up -d`) |

## Layout

```
Cargo.toml              workspace root
crates/api/             HTTP server (binary: circular-api)
crates/api/templates/   Askama templates and HTMX partials
crates/worker/          (planned) background job consumer for Redis Streams
migrations/             SQLx migrations, shared by all crates
docker-compose.yml      Postgres + Redis for local development
```

## Data ownership

- **PostgreSQL is the source of truth.** Users, workspaces, boards, issues, comments and memberships all live here.
- **Redis only holds data that is ephemeral, derived, or in transit:** sessions, rate-limit counters, pub/sub messages, job streams and caches. Losing Redis should log users out and drop in-flight events, but must never lose domain data.

## Roadmap

Work proceeds milestone by milestone. Do not start a milestone until the previous one is merged.

1. **Skeleton**: workspace, Compose, `/health` *(done)*. Extend `/health` to check Postgres and Redis.
2. **Issues CRUD**: `issues` table, SQLx pool in shared `AppState`, JSON endpoints.
3. **Error handling**: `AppError`, `?` throughout handlers, consistent error responses.
4. **Auth**: users table, registration and login, argon2, Redis-backed sessions, `CurrentUser` extractor.
5. **Rate limiting**: Tower middleware backed by Redis counters on auth and write endpoints.
6. **Workspaces and boards**: memberships, per-resource authorization checks.
7. **Tests**: integration tests with `#[sqlx::test]` covering handlers and authorization.
8. **Frontend**: Askama layouts, board view, HTMX partials for create, edit and status changes.
9. **Real-time**: Redis Pub/Sub per board, fanned out to clients over SSE or WebSockets; must work with multiple API instances.
10. **Stretch**: `worker` crate consuming Redis Streams (notifications), read-through caching where measurements justify it.
11. **Deploy**: Dockerfile, CI (fmt, clippy, test), hosted deployment.

## Implementation ownership

This codebase is built jointly by the maintainer and the agent. The maintainer is expected to understand and own every part of it, so implementation work is deliberately split.

**The agent implements:**
- The **first instance of every pattern**: the first handler, the first query, the first extractor, the first template or partial, the first test.
- Infrastructure and cross-cutting code: `AppState`, `AppError`, middleware, the session layer, the Pub/Sub fan-out, configuration, Compose, CI and Docker.
- Migrations, when the schema change is part of a milestone the agent is scaffolding.

**The maintainer implements:**
- **Subsequent instances of an established pattern.** Once `issues` has CRUD handlers, `comments` handlers are the maintainer's. The same applies to additional queries, templates, partials and tests.
- Any code the agent marks `TODO(human)`.

**How to hand off work:**
- Leave a compiling stub (use `todo!()` where needed) with a `// TODO(human): ...` comment that states what to build, which existing code to model it on (`see crates/api/src/issues.rs::create`), and any non-obvious constraints (authorization rule, SQL edge case, HTMX swap target).
- Keep each `TODO(human)` small and self-contained: one function, one query, one template.
- List all open `TODO(human)` items at the end of your response.
- **Never complete a `TODO(human)` yourself** unless the maintainer explicitly asks. When asked for help with one, give hints, point to the reference implementation, or explain the compiler error. Do not write the solution unless it is requested.
- When the maintainer finishes a `TODO(human)`, review it for correctness and idiomatic Rust, and explain any requested changes rather than silently rewriting them.

## Working agreements

- **Plan before code.** For each milestone or feature, first outline the approach, the files involved, and the Rust or library concepts that will appear (e.g. custom extractors, `Arc` shared state, `tokio::select!`). Wait for confirmation before implementing.
- **Small changesets.** One feature per change, roughly 50–150 lines of non-generated code. Split anything larger.
- **Explain non-obvious code.** After implementing, summarize the key design decisions and any Rust idioms that are not self-evident (lifetimes, trait bounds, `Send + 'static` requirements, error conversions). Keep it brief and specific to the diff.
- **No unrequested dependencies.** Propose new crates before adding them, with a one-line justification.
- **Don't skip ahead.** Do not implement features from later milestones, even if convenient.

## Conventions

- Raw SQL with `sqlx::query!` / `query_as!`; no ORM. All schema changes go through `sqlx migrate add <name>`, never manual DDL.
- Handlers return `Result<impl IntoResponse, AppError>`. No `unwrap()` / `expect()` in request paths; they are acceptable only at startup.
- Configuration comes from environment variables (see `.env.example`). No hard-coded secrets.
- HTMX endpoints return partials; full-page routes render a layout. Keep partial templates next to the page that uses them.
- Before declaring work done: `cargo fmt`, `cargo clippy --all-targets -- -D warnings`, `cargo test`.

## Commands

```sh
docker compose up -d                 # start Postgres and Redis
cargo run -p circular-api            # run the server on BIND_ADDR (default 127.0.0.1:3000)
sqlx migrate run                     # apply migrations (requires sqlx-cli)
cargo test                           # run tests (requires Postgres running)
```
