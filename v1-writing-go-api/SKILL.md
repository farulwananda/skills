---
name: fw-writing-go-api
description: Use when building, reviewing, or operating Go HTTP/REST APIs, including net/http or Chi routing, project-selected databases such as MySQL or PostgreSQL, GORM or SQL persistence, HTTP contracts, authentication, transactions, production lifecycle and deployment readiness. Keep architecture proportional; pair with fw-writing-go for language and concurrency rules.
license: MIT
metadata:
  author: Farul Wananda Go API conventions
---

# FW Writing Go API

## Scope and first inspection

Use for Go HTTP APIs and their production boundaries. Use [Go language guidance](../fw-writing-go/SKILL.md) for errors, concurrency and ownership; this skill does not replace it. Not a mandate to rewrite a CLI, replace a working router/ORM, introduce microservices, or deploy anything.

Inspect `go.mod`/`go.sum`, Go/toolchain and CI versions, actual router/driver APIs, application bootstrap, routes/middleware, auth/proxy topology, database schema, tests and deployment runbook before editing. Establish the endpoint's request/response contract, permissions, side effects, deadlines and compatibility requirements. Never infer business requirements from the words "production" or "enterprise".

For a new application, use a security-supported stable Go line at its current patch, pinned in CI/builds. Preserve the supported range of an existing repository unless migration is authorized. Modern method/path `ServeMux` patterns require Go 1.22+; this syntax minimum is not a recommendation to deploy an unsupported release. Do not silently upgrade dependencies or execute downloaded installers.

## Recommended stack: choose only needed capabilities

These are defaults for a greenfield REST service, not a required dependency bundle. Existing compatible repository conventions win.

Choose the database and data-access style independently from project requirements, existing infrastructure, team familiarity and explicit user preference. Do not silently default to PostgreSQL, MySQL, raw SQL or an ORM. GORM is a first-class choice for new or existing projects when an ORM fits; it is not merely a legacy exception.

| Capability | Default decision | Add or change only for a concrete need |
| --- | --- | --- |
| HTTP routing | Standard `net/http` with a private `ServeMux` | Chi v5 for useful groups, composable middleware and subrouters; retain `http.Handler` compatibility. Keep existing Gin/Echo when sound; do not migrate for style. |
| Persistence, if required | The project's selected engine: MySQL and PostgreSQL are both valid production choices | Match driver/dialect, supported server version, hosting, operational skills and feature requirements. SQLite is appropriate when its deployment/concurrency model fits. A stateless API needs no database. |
| Data access | GORM when an ORM fits the project; `database/sql` or a suitable native driver when SQL-first fits | MySQL + GORM uses `gorm.io/driver/mysql`; PostgreSQL + GORM uses `gorm.io/driver/postgres`. Use pgx only for PostgreSQL. sqlc is optional with the selected engine/generator, not a mandatory companion to GORM. Do not add parallel access stacks without a real need. |
| Schema changes | The existing reviewed migration workflow; Goose SQL migrations if selecting a new tool | Match the selected database and deployment process. GORM is valid without making `AutoMigrate` a per-replica production-startup step; never auto-migrate/reset an unverified target. |
| Configuration/logging | Validated environment configuration and standard `log/slog` | Existing settings/logger packages if useful; no automatic Viper, Zap or framework config layer. |
| Authentication | Existing session/OIDC/identity-provider contract | Select transport and trust model explicitly. JWT is a format, not a default auth architecture; never write custom cryptography. |
| Verification | `testing`, `httptest`, focused race checks and actual local HTTP smoke | Disposable integration coverage using the selected database engine/version for DB-specific behavior; reuse existing tooling. |
| Delivery | One binary on an appropriate managed service or VPS; container if the host uses it | Managed MySQL/PostgreSQL or another selected datastore when the team needs it; no default Kubernetes, service mesh or multiple services. |

Redis, queues, OpenTelemetry, Prometheus and OpenAPI tooling are optional. Add cache only with freshness/invalidation evidence, durable jobs only with a delivery requirement, and telemetry with an actual operating/collection plan. A public or shared API should have an explicit versioned contract; choose existing OpenAPI tooling when it benefits real consumers, not a UI dependency by reflex.

## Architecture proportional to the domain

YAGNI and KISS are the baseline. Consider domain complexity, data/financial risk, integrations, operational resources and team ownership—not just request volume or file count.

- A small service may keep cohesive handlers and queries in a few files. A database call in a handler is not automatically a violation.
- Add a service/action for a business workflow, reusable policy or transaction owner, not to forward every route call.
- Use feature packages when domain boundaries become real. Prefer a modular monolith over distributed deployment unless independent deployment or failure isolation is required.
- Apply Clean Architecture through meaningful dependency direction: business decisions should not depend unnecessarily on HTTP details. Do not impose a four-layer folder template or hide every SQL call behind an interface.
- Use strategic DDD language/ownership where it clarifies the business. Add aggregates/value objects/domain services only for actual invariants; do not duplicate persistence models just to match a diagram.
- Interfaces belong near consumers when substitution or a stable capability boundary is needed. One implementation can still justify an interface; one interface per struct is not required.
- Preserve useful existing abstractions and public contracts. Simplifying means removing needless complexity, not deleting authorization, transaction or provider safeguards.

## HTTP contract and security

- Keep request DTOs and explicit public response fields separate from internal/database shapes where they differ. Validation never grants authorization.
- Limit request bodies with `http.MaxBytesReader`, bound headers/list sizes/uploads, check content type, and define empty/null/malformed/multiple-value JSON behavior. `Content-Length` alone is not enforcement. Use unknown-field rejection only for an intentionally closed contract; `encoding/json` does not reject duplicate keys by default.
- Encode/validate output before writing headers when serialization can fail. Once committed, do not attempt to replace the status with a fresh JSON error. Streaming needs its own failure/cancellation contract.
- Define status/error semantics consistently: malformed input 400, unauthenticated 401, unauthorized 403, not found 404, conflict 409, payload too large 413, unsupported media type 415, known dependency failures 502/503/504 as appropriate, unexpected server errors 500. Preserve the established contract rather than blindly standardizing it.
- Keep errors public-safe; retain useful causes in redacted server diagnostics. Do not log tokens, DSNs, cookies, authorization headers, raw private payloads or SQL arguments. Bound client-supplied correlation IDs and use route templates, not secret/high-cardinality URLs, for telemetry.
- Enforce authorization and tenant/owner scope on every read/write, not only at router-group registration. Test negative/sibling routes. Use parameterized SQL; allowlist dynamic SQL identifiers and sort keys.
- Same-origin APIs may need no CORS. Cross-origin credentials need an explicit allowlist and deliberate cookie/CSRF policy. CORS is not authentication. Trust forwarded identity/IP/host headers only from configured ingress; do not blindly trust Chi RealIP-style middleware.
- Authenticated/private responses must bypass shared cache reads/writes and follow the intended `private, no-store` policy. A URL-only cache or `Vary` alone does not establish tenant isolation.
- Reuse appropriate secure headers and the host's TLS configuration. Do not expose pprof, metrics, admin endpoints or detailed dependency errors publicly by default.

## Runtime, data and production ownership

Read [Production decisions](references/production.md) before changing persistence, auth, lifecycle, deployment or operational guarantees. It covers pool budgets, transaction/idempotency ownership, deadlines, shutdown, readiness, testing and release controls.

Use [Runnable HTTP reference](references/http-example.md) only when a concrete net/http example helps. It is a complete local public arithmetic API with bounded JSON input and signal-driven drain, not an authenticated business application or universal production blueprint. Do not paste it over an existing bootstrap.

No automatic dependency installation, external provider calls, database resets, migrations against unverified targets, provisioning, deployment, DNS changes or pushes. For remote mutation, establish the exact target and applicable authorization first.

## Verification and delivery

Inspect verification scripts/bootstrap for side effects. Run the focused configured checks and execute the changed HTTP path locally. Cover valid and invalid inputs, method/route behavior, auth/tenant isolation, error status/public-detail safety, transaction rollback and duplicate side effects where relevant. Test bounded concurrency/cancellation rather than assuming a timeout stops work.

Before DB tests, prove the connection and harness are disposable and worker-isolated. An environment named `test` is not proof. SQLite does not establish MySQL/PostgreSQL dialect, collation, constraint, locking or migration behavior; exercise the chosen engine where those contracts matter. Never reset development or production data to make tests pass.

For lifecycle changes, exercise an in-flight request through SIGTERM, listener closure, drain completion and resource teardown; compilation and `httptest` alone do not prove signal handling. For delivery, verify the actual target runtime, probes, TLS/proxy behavior, migration compatibility, rollback and backup/restore as applicable—never deploy just to verify the skill.

Report exact commands/versions exercised, assumptions and unverified DB/provider/host behavior. Passing a sample is not certification of production readiness, throughput, security or SLA.

## Sources and provenance

Original project guidance, created 2026-10-15; independent synthesis, not copied third-party skill text. Consult installed-version documentation selectively:

- [Go release/support policy](https://go.dev/doc/devel/release), [net/http](https://pkg.go.dev/net/http) and [Go 1.22 routing changes](https://go.dev/blog/routing-enhancements).
- [Chi](https://github.com/go-chi/chi): optional router composed with standard HTTP handlers.
- [GORM database drivers and pools](https://gorm.io/docs/connecting_to_the_database.html), [transactions](https://gorm.io/docs/transactions.html), [updates](https://gorm.io/docs/update.html): MySQL/PostgreSQL are supported choices; match installed APIs.
- [pgx](https://github.com/jackc/pgx) for PostgreSQL only, [sqlc transactions](https://docs.sqlc.dev/en/stable/howto/transactions.html) when SQL generation is selected, [Goose](https://pressly.github.io/goose/).
- [Structured logging with slog](https://go.dev/blog/slog). More detailed operational sources are linked in the production reference.
