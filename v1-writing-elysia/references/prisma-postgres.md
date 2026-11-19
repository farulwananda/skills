# Prisma and PostgreSQL Adapter

Read this reference only when Prisma ORM or PostgreSQL is in scope. Verify the installed Prisma major against current official documentation before choosing generators, imports, adapters, or configuration; these changed materially between major versions.

## Dependency Contract

Prisma is relevant when the project has already selected it, or the task explicitly selects persistence. Inspect the installed major, generated client, schema/config, adapter, pool ownership, and runtime before changing packages; do not install, generate, migrate, or provision a database automatically.

The **versioned Prisma 7** Bun/PostgreSQL guide describes this path (not a current-major recommendation):

- `prisma` as a development dependency;
- aligned `@prisma/client` and `@prisma/adapter-pg` versions;
- `pg` plus its development types;
- ESM and the `prisma-client` generator with an explicit output path;
- the datasource URL in `prisma.config.ts`, while the runtime adapter also receives the connection string.

Do not copy this list to another major or a different driver. In particular, current unversioned Prisma docs may describe a different client/transaction API. Follow the installed major's docs and generated types; do not upgrade to match a redirected page. Keep CLI/client/adapter versions mutually supported in the lockfile.

Prismabox is not a default dependency. Elysia's supported wire schemas own API validation while Prisma owns persistence types. Generate API schemas from database models only when deliberate coupling is desired and field visibility, nullability, serialization, and compatibility have been reviewed.

## Client Lifetime

In a long-running process, reuse the project-owned client and pool rather than constructing them per request. Scope this recommendation to the actual Prisma major and adapter: some clients create a pool, some accept an externally owned pool. Do not create a second pool accidentally or `$disconnect()` after every handler.

The database owner constructs and closes only the resources it owns. A route/plugin importing a shared client must not disconnect it on its own. Tests/CLI scripts close the clients/pools they create; externally provided pools may require explicit separate shutdown according to adapter ownership options. Hot-reload reuse follows the repository's existing convention, not an unconditional new global.

Bound connection capacity across replicas × workers × pools, leaving headroom for migrations and administration. Use the hosting platform/driver's supported edge or serverless strategy; a Bun/PostgreSQL connection model is not automatically valid on Workers.

When health/shutdown behavior is in scope, keep readiness queries bounded and liveness independent of database availability. The process owner stops new traffic and drains in-flight work before closing owned resources. Do not add lifecycle hooks just to duplicate an existing owner.

## API and Database Boundaries

Define Elysia request and response schemas independently from Prisma models.

- Select only fields the API is allowed to expose.
- Map `Date`, `Decimal`, `BigInt`, bytes, enums, and nullable fields explicitly.
- Never return password hashes, tokens, internal moderation fields, provider payloads, or raw Prisma errors.
- Keep response compatibility stable when database columns change.
- Treat generated Prisma types as internal persistence types, not DTOs.

For a simple query, a route or service may call Prisma directly. Add a focused query/store module when several routes reuse a query or when it materially improves query review. Do not create a generic base repository or a one-to-one interface over every Prisma method.

## Schema and Query Design

Design constraints from real invariants:

- primary keys and foreign keys;
- unique slugs, idempotency keys, or other true uniqueness rules;
- intentional nullability and deletion behavior;
- indexes matching actual filter, join, and ordering prefixes;
- deterministic ordering for pagination;
- bounded `take`/page size and cursor pagination for growing lists.

Avoid convenience-driven `include` trees; use `select`, bounded relations, and aggregation to limit leakage, N+1 queries, and payload size. Do not assume an ORM query shape alone proves its query count. Inspect plans/query behavior when the task reveals a performance problem, before introducing caching. Parameterize raw SQL; never concatenate untrusted identifiers/values into unsafe query APIs.

## Transactions and Concurrency

Use a transaction only when changes form one atomic invariant; a single atomic write needs no wrapper. The operation owner starts it and passes its transaction-scoped client to every helper. For Prisma 6/7 interactive transactions this is the callback's `tx` from `prisma.$transaction`; use that major's generated transaction type. Other majors may expose a different API. Calling the root client inside a callback escapes that transaction; a helper starting a second transaction does not make it nested.

Keep transactions short, using supported timeouts/isolation for the installed major and database. Never hold one open for email, storage, payment, or another network call. Commit before external effects, and define recovery only if required; an HTTP failure after commit must not invite unsafe whole-operation replay.

Handle concurrency with database guarantees rather than read-then-write assumptions:

- unique constraints for duplicate prevention;
- atomic update conditions for state transitions;
- idempotency keys for retryable writes;
- appropriate isolation or row locking only when the invariant requires it.

Map only recognized installed-version errors/constraints to stable outcomes: for example, a known unique business key can be 409, a missing target can be 404, while database unavailability is not either. For Prisma 6/7, check the generated/runtime known-error type and code (such as `P2002`) rather than message substrings. Distinguish which constraint failed; do not relabel every database error as conflict.

Retries are not a default requirement. If explicitly needed, restrict bounded retries to documented transient database failures and a retry-safe whole transaction. Do not replay validation, permission, or uniqueness failures, and never replay payment/email/storage/provider work inside a database retry. Idempotency requires a durable scoped key and a defined result/duplicate policy, not merely catching a duplicate error.

Add an outbox only when an external side effect must be durably retried after an atomic database commit. A basic CRUD or read-only portfolio API does not earn a queue or outbox by default.

## Migration Safety

- Inspect the active environment and target database before any migration command.
- Generate/review migrations when requested; applying them requires an authorized target/deployment step, never application startup or incidental verification.
- Prefer additive, backward-compatible expand/migrate/contract changes for live data.
- Never reset, wipe, drop, truncate, or accept an interactive reset without fresh explicit user authorization.
- Test destructive-looking schema changes against isolated disposable data first.
- Seed scripts must be deterministic or intentionally idempotent and must never contain real credentials.

## Testing

When database checks are authorized, use an isolated PostgreSQL database/schema with a dedicated verified connection string and test-owned cleanup. Verify:

- constraints and relations, not mocked Prisma call counts;
- exact selected response fields;
- not found and uniqueness conflicts;
- pagination boundaries and deterministic ordering;
- rollback, transaction-client propagation, resource ownership, and duplicate/idempotency behavior where required;
- migration application from an empty database and, for risky changes, from representative prior schema/data.

Do not point tests at an unverified development or production database. A testing environment name alone does not prove isolation.

## External Dependencies Beyond Prisma

Before adding cache, queue, logger, rate limiter, schema generator, or database helper, record:

1. the missing native/Elysia/Prisma capability;
2. why a local function, PostgreSQL, or platform ingress cannot meet it;
3. state and consistency behavior across replicas;
4. maintenance, license, security history, Bun compatibility, and transitive cost;
5. removal or failure-mode implications.

If those answers are absent, keep the dependency out.

## Official Sources

- [Prisma with Bun](https://www.prisma.io/docs/guides/v7/runtimes/bun)
- [Prisma database drivers](https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/database-drivers)
- [Connection management](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/connection-management)
- [Connection pooling](https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/databases-connections/connection-pool)
- [Versioned Prisma 7 documentation](https://www.prisma.io/docs/orm/v7): use this when the lockfile selects 7; inspect transaction and adapter ownership docs for that release.
- [Current transaction documentation](https://www.prisma.io/docs/orm/fundamentals/transactions): checked for ownership principles only; current APIs must not be substituted into an older client.

The versioned Bun guide and transaction ownership guidance were consulted on 2026-10-15. No live database, migrations, shutdown, or retry behavior was exercised by this documentation audit. These are boundary rules, not a runnable client setup or a guarantee across Prisma majors.
