---
name: fw-writing-elysia
description: Use when writing, reviewing, or refactoring Elysia TypeScript backends, including version-compatible routes, plugins, lifecycle hooks, response schemas, Eden clients, runtime adapters, or Prisma and PostgreSQL persistence.
---

# FW Writing Elysia

## Overview

Build repository-native Elysia backends while preserving inference. Choose the smallest boundary that owns real behavior; do not impose a modular monolith, dependency, or infrastructure on a small service or an existing architecture.

Use for Elysia HTTP/plugin boundaries, not generic TypeScript, React UI, standalone database administration, or deployment changes. Bun is a common host, not a universal runtime assumption; inspect the actual adapter before using Bun-only facilities.

## Workflow

1. Inspect `package.json`, lockfile, installed Elysia/plugin/Prisma versions, nearby modules, runtime adapter, deployment shape, and existing checks. Use matching version docs and exported types; do not automatically migrate APIs to current documentation.
2. State the contract: method/path, input and status-specific output schemas, authentication/authorization, persistence effects, and bounded queries.
3. Reuse Web/runtime APIs, Elysia core, and existing project capabilities. Select an official compatible plugin only for a real gap; do not install dependencies automatically or require a new validator, logger, cache, retry layer, or architecture.
4. Keep HTTP concerns in a route/module. Use a feature Elysia instance when a boundary is useful; add shared schemas or services only when reuse, business behavior, or a transaction earns them.
5. Validate consumed untrusted input and wire outputs. Use the installed typed status API for expected failures; distinguish malformed input, validation, authentication, authorization, not-found/conflict, and unexpected server errors. Never expose internal error details or turn every failure into 200/500.
6. Register hooks deliberately with their installed scope and order. Test policy coverage on composed routes, including negative/sibling cases; a hook in an auth plugin is not automatically global.
7. Use `app.handle(new Request(fullUrl))` or the project's Eden/test integration, then focused type/tests and proportionate build checks when authorized. Verify the deployed adapter separately when relevant. Report only exercised checks and limits.
8. Do not deploy, push, reset/apply database changes, or call paid providers automatically. Do not retry paid/non-idempotent side effects without an explicit authorized idempotency and retry contract.

## Optional Feature Layout

```text
src/
  app.ts                 # compose plugins/modules; export App type
  index.ts               # validate startup, listen, shutdown
  modules/<feature>/
    index.ts             # Elysia routes and HTTP concerns
    model.ts             # only when schemas are reused
    service.ts           # only when business logic earns it
```

Adapt this layout only if the repository needs it. A shared database owner can be necessary for connection/lifecycle safety even with one consumer; it does not require a new layer stack. Do not pre-create controller, repository, use-case, DTO, mapper, event, queue, or cache layers.

## Dependency Gate

| Need | Default |
|---|---|
| Request/response validation | Existing supported schemas; Elysia `t` when none is established |
| Cross-origin requests | Official `@elysia/cors`, only for actual cross-origin clients |
| API contract publication | Official `@elysia/openapi`, only when a consumer needs it |
| Type-safe first-party client | Official `@elysia/eden`; keep client/server Elysia versions aligned |
| Authentication | Decide session vs token first; then use the relevant official capability |
| Tracing | Official OpenTelemetry only when a collector and trace use-case exist |
| Persistence | Project-selected database library; read the Prisma reference when applicable |

The package names above follow current docs; preserve an older compatible official namespace if installed. An external package needs a real capability gap plus maintenance, Elysia/adapter/runtime compatibility, security-surface, and simpler-alternative review. “Enterprise”, “scalable”, familiarity, deadlines, and product labels are not capability gaps. Do not infer auth, documentation consumers, persistence, or asynchronous work from them.

## References

- Read [references/elysia-patterns.md](references/elysia-patterns.md) for routes, schemas, lifecycle, errors, context, Eden, and tests.
- Read [references/official-ecosystem.md](references/official-ecosystem.md) before selecting plugins, authentication, logging, or security dependencies.
- Read [references/prisma-postgres.md](references/prisma-postgres.md) only when Prisma or PostgreSQL is in scope.

Primary docs and public-skill provenance are linked in those references. They are independently synthesized guidance, not copied upstream skills or evidence that an uninspected installed version supports an API.

## Done When

The change preserves inference, implements the validated contract and policy scope on the actual runtime, keeps handlers/transactions bounded, adds no speculative layer or dependency, and avoids secrets/internal-error leakage. State relevant checks actually run and unresolved environment limits.
