# Go API Production Decisions

Read only the sections relevant to the task. This is operational decision support, not permission to provision services, run migrations, connect to production or add every listed mechanism. Match the installed Go, driver and host versions.

## Deployment shape and stack selection

A small production API can be one Go process, the host's ingress/TLS layer and a database only if persistence is required. For a greenfield REST service, start with net/http, slog and validated configuration. Add Chi v5 when grouping/mounting/middleware composition earns a router dependency. Gin/Echo are valid established choices; avoiding a migration is usually safer than standardizing routers.

Select the database from actual project requirements and user preference, not from a universal PostgreSQL recommendation. MySQL and PostgreSQL are both production options; consider existing infrastructure, team knowledge, required features, server versions and operating costs. SQLite can be appropriate for a suitable single-node/embedded workload. Do not switch engines for style or assume their SQL, isolation, collation, DDL and error codes are interchangeable.

Select data access separately. GORM is a first-class ORM choice for a new or existing MySQL/PostgreSQL application when its model/query workflow fits. Use the corresponding `gorm.io/driver/mysql` or `gorm.io/driver/postgres`; do not introduce pgxpool for MySQL or alongside GORM merely to satisfy a template. SQL-first code can use `database/sql` with the matching driver, or pgx/v5 for PostgreSQL. sqlc is optional for a growing SQL surface and must target the selected engine/generator. Neither raw SQL nor an ORM is universally required; preserve an established sound access strategy.

A Go net/http binary needs a host that runs Go processes/containers. Standard Cloudflare Workers/Pages configuration does not host that binary; using Cloudflare DNS/proxy does not change the application's runtime. Choose an appropriate PaaS/container service or VPS first. Do not assume WebAssembly is a drop-in replacement for a long-lived net/http server.

## Database and transaction ownership

- Create one shared pool per intended process/database owner, not one per request. Parse validated configuration; confirm required connectivity with a bounded startup/readiness check. Pool construction alone is not evidence of reachable credentials/schema.
- Budget connections across replicas, background workers, rolling-deploy overlap, migration jobs and database limits. Bound acquisition/query time through context and driver settings; an unlimited queue waiting for a small pool can still exhaust service memory.
- Share the owned pool, not a mutable request/query chain. For GORM, configure the underlying `database/sql` pool obtained through `db.DB()` where supported, handle its error, and close it at its actual owner. For native pgx, use pgxpool only with PostgreSQL. An acquired connection/transaction needs a clear owner; do not assume it is safe for concurrent queries. Close rows, inspect row/iteration errors, and release acquired connections on every path.
- Bind values, allowlist dynamic identifiers, select public fields and bound result sets. Use stable ordering and suitable pagination. Check query plans/indexes on representative data rather than adding caching first.
- Choose one transaction owner per business unit. Helpers must use that transaction: GORM's `tx` inside `Transaction`, sqlc's generated `WithTx` where supported, or the selected driver's transaction handle. Do not silently execute through the root DB/pool or commit independently. Check transaction/commit errors before reporting success. Driver cancellation semantics do not remove explicit cleanup ownership; in particular, a native pgx begin context is not automatic rollback.
- Use database constraints/atomic updates/appropriate locking or isolation for concurrent invariants. Application validation before a write does not enforce uniqueness, stock or balances under races. A generated query is not proof that a read-modify-write sequence is safe.
- A database transaction does not roll back an external payment/provider request. On uncertain outcomes, use the existing provider idempotency/status-reconciliation contract, not automatic resubmission. Local mutexes do not provide crash-safe or cross-replica deduplication.
- Retry only documented transient, safe operations within a bounded budget. Retrying a transaction callback must not replay paid/external side effects. Add an outbox only for a real durable-delivery requirement; it is not standard CRUD scaffolding.

## GORM-specific safeguards

GORM is an accepted implementation choice, not a reason to create repository/interfaces around every model. Follow the installed API style: traditional GORM and newer generics APIs differ; do not mechanically migrate code or assume both exist.

- Carry request context through the supported API, such as `WithContext(ctx)` in traditional GORM. Start each request/query from an appropriate clean session; do not share a mutated chain and accidentally retain another request's filters.
- Scope queries/updates/deletes by authorized tenant/owner and trusted identity. Map request fields explicitly; do not bind untrusted input directly into a persistence model or turn on global updates to bypass missing conditions.
- Traditional struct `Updates` omits zero values by default. For intended `false`, `0` or empty-string changes, use an explicit writable-field map or field selection while preserving PATCH omitted-versus-present semantics. Do not use `Save` as a generic scoped update: its upsert behavior can create an unintended row.
- Check the actual returned error and affected-row semantics for the selected driver/API. Preserve soft-delete, hooks, associations and error translation intentionally. Avoid unbounded association loading/N+1 queries; inspect generated SQL and indexes before adding cache or raw SQL.
- A default transaction around one GORM write is not a transaction around an entire multi-step business workflow. Use an explicit owner for that unit, operate through its `tx`, and preserve paid-operation/idempotency safeguards. Do not disable default transactions solely on a generic performance claim.
- Use the selected driver's connection, TLS, encoding and time-handling configuration. For MySQL, verify relevant `utf8mb4`/`parseTime` and timezone requirements; never copy example credentials or insecure TLS settings. PostgreSQL-specific configuration is not a MySQL template.


## Migrations and recoverability

Keep one reviewed migration workflow suited to the selected database. Retain an existing GORM-based or SQL-based release process when safe; Goose SQL migrations are an option when selecting a new tool, not a requirement to replace it. GORM `AutoMigrate` is not a substitute for reviewing production DDL, compatibility, locks or rollback, and must not run automatically on every replica or an unverified database. Tool installation/version changes still require the normal dependency decision.

Review the exact connection and privileges before mutation. Apply migrations to a verified disposable database first. Do not default to reset/down/drop commands or allow every app replica to race schema changes at startup. Use a controlled release step, compatible expand/contract changes where rolling deployment needs them, and a plan for long locks, failed/non-transactional operations and old/new binary overlap.

Do not equate a down migration with safe rollback: destructive changes may need restoration or forward repair. Establish backup ownership and exercise restoration on an isolated target when data matters. Never imply that a successful local migration verifies production permissions, scale or restore time.

## HTTP budgets and cancellation

- Use an explicit `http.Server`, private mux and reviewed `ReadHeaderTimeout`, body-read policy, `WriteTimeout`, `IdleTimeout` and `MaxHeaderBytes`; convenience `http.ListenAndServe` with defaults is not a complete timeout policy. Header limits do not bound body size.
- Derive request-level deadlines from actual upstream/DB work and the host's timeout budget. `WriteTimeout` limits socket writes; it does not terminate a handler. Context cancellation is cooperative, not a guarantee that CPU work, a blocking library or a remote side effect stopped.
- Pass request context into the chosen database driver/ORM, outbound requests and owned work. Bound concurrent expensive work and queue/admission depth; default net/http request handling plus timeouts does not establish a global concurrency bound. Avoid detached request goroutines.
- For outbound HTTP, reuse an owned client/transport with appropriate connection, response-header, body and overall budgets. Bound response bodies and close them. A transport success can still carry an HTTP failure status. Validate destinations and redirect behavior for user-provided URLs to prevent SSRF.
- Streaming/SSE, uploads and WebSockets need tailored deadlines/backpressure/cancellation. A short blanket `WriteTimeout` can break legitimate streams. Confirm `Flusher`, `Hijacker`, `Pusher` or `ResponseController`/`Unwrap` support before introducing middleware wrappers; a status-logging wrapper can accidentally break host capabilities.
- Middleware that observes `ctx.Done()` is not a worker kill switch. Never race a timeout response against an unsynchronized handler writing to the same ResponseWriter. Once headers/body are sent, failures must follow the streaming/connection contract, not a second status response.

## Authentication, exposure and private data

Reuse the application's identity system. For OIDC/JWT, use maintained existing validation tooling, an explicit trusted issuer/audience/algorithm/key policy and expiry/rotation handling; decoding a token is not verification. For browser sessions, require intentional Secure/HttpOnly/SameSite, CSRF and logout/revocation behavior. Do not infer a need for JWT, OAuth, refresh-token storage or an identity service from the word API.

Apply resource/action authorization and tenant scope in the actual read/write path. Rate limiting belongs at trusted ingress or an appropriate shared enforcement boundary when replicas share a quota; an in-memory limiter is per-process, not globally authoritative. Configure trusted proxy hops before trusting forwarded IP/host/scheme fields.

Log safe structured events through the existing logger/slog owner. Redact credentials, DSNs, tokens, cookies and private payloads. Avoid high-cardinality user/path labels in metrics. Public errors must not reveal SQL, stack traces or sensitive existence/ownership details. Expose diagnostics/pprof only through a deliberately protected administration boundary; do not mount the global default mux on a public listener by accident.

## Liveness, readiness and shutdown

Liveness answers whether the process can run, not whether every dependency is healthy. Readiness answers whether this instance should accept its intended traffic; check only required capabilities with bounded/cached probes as appropriate. Returning a hardcoded 200 for a DB-dependent service is not evidence of usable schema/connectivity. Do not restart the fleet because a shared database is briefly unavailable.

The bootstrap owns listener, pools, clients and background workers:

1. Arrange signal handling and startup failures so owned resources are cleaned up and errors are returned to main.
2. On drain, mark unready and stop accepting traffic; account for ingress propagation according to the host's termination budget.
3. Call `Server.Shutdown` with a fresh, finite cleanup context. It must not inherit an already-cancelled signal/request context. Do not cancel the base request context first if the goal is to let in-flight work finish.
4. Wait for Shutdown itself. `Serve` returning `ErrServerClosed` happens early and does not prove drain completion.
5. Separately notify/join hijacked WebSockets and application workers; Shutdown does not own them and `RegisterOnShutdown` callbacks are not an automatic join barrier.
6. Close pools/clients after their users have finished. On timeout, follow an explicit force-close/cancellation policy, report failure and return nonzero. Closing sockets does not prove arbitrary goroutines terminated; the supervisor needs a final termination bound.

Use a returning run function so cleanup completes before main selects an exit code. Second-signal behavior and readiness propagation should match the deployment, not a fixed copied delay.

## Build, release and evidence

Use a supported Go line at its latest reviewed patch. Pin the compiler, dependency versions and build/runtime images through the project's release policy. Run proportionate focused checks, `go vet`, race checks where supported and `govulncheck` when the existing toolchain provides it; do not auto-install scanners or rewrite CI for an unrelated endpoint change.

For container delivery, use a multi-stage build where useful, a non-root runtime, minimal writable paths and proper signal forwarding. A static binary depends on the actual CGO/native dependencies; do not assume `CGO_ENABLED=0` works for every application. Supply CA roots for outbound TLS and other runtime data only when required. Do not disable certificate verification to make a minimal image work. Keep secrets out of source, build args, image layers and logs. Apply resource limits and an existing restart policy; no automatic Kubernetes requirement.

Verify endpoint behavior with real local HTTP in addition to handler tests. Use an isolated instance of the selected database engine/version for constraints, transaction/rollback, migrations, dialect/collation/locking and tenant isolation; prohibit shared/production resets. SQLite or query-generation-only tests do not prove MySQL/PostgreSQL server semantics. Exercise SIGTERM with an in-flight request and observe listener closure, drain completion, background resource closure and the exit result. Load-test only when capacity is in scope and the target is safe; routing benchmarks do not prove application throughput or production readiness.

## Database policy verification

On 2026-10-15, four instruction-level smoke probes exercised explicit MySQL + GORM selection, preservation of existing PostgreSQL + pgx/sqlc, a tenant-scoped GORM PATCH including `false`, and a stateless API needing no database. The resulting approaches retained the chosen stack, avoided forced parallel dependencies and preserved isolation/migration safeguards. These probes check guidance behavior, not database execution: no MySQL/PostgreSQL integration or GORM runtime was exercised in this policy revision.

## Primary references

Author synthesis, recorded 2026-10-15; match documentation to installed versions:

- [net/http server and Shutdown](https://pkg.go.dev/net/http), [context](https://pkg.go.dev/context), [Go support policy](https://go.dev/doc/devel/release).
- [GORM database drivers and pools](https://gorm.io/docs/connecting_to_the_database.html), [transactions](https://gorm.io/docs/transactions.html), [update semantics](https://gorm.io/docs/update.html). GORM supports both MySQL and PostgreSQL; examples do not authorize automatic migration or dependency installation.
- [GORM context propagation](https://gorm.io/docs/context.html) and [method chaining/session reuse](https://gorm.io/docs/method_chaining.html): follow installed traditional/generics APIs and avoid cross-request condition pollution.
- [pgx](https://github.com/jackc/pgx), [pgxpool](https://pkg.go.dev/github.com/jackc/pgx/v5/pgxpool) for PostgreSQL, [sqlc transactions](https://docs.sqlc.dev/en/stable/howto/transactions.html) for the selected generator. Generated transaction helpers bind an owner; they do not define business isolation guarantees.
- [Goose documentation](https://pressly.github.io/goose/) for the selected tool/version, not authorization to run its commands.
- [slog](https://go.dev/blog/slog), [govulncheck](https://go.dev/doc/tutorial/govulncheck). These recommendations do not mandate new logging or scanning dependencies.
