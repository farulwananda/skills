# Official Elysia Ecosystem and Fallbacks

Official plugin catalog, CORS/Eden package pages, and Node/Cloudflare integrations consulted on 2026-10-15. This is a capability index, not a default dependency list or a guarantee for older releases. Before a dependency change, inspect installed versions, lockfile, matching docs, peer constraints, and the deployment adapter. Do not install packages automatically.

Current docs use `@elysia/*`; older repositories may use another official namespace. Keep the installed compatible set unless migration is requested; never rename imports merely because a current page differs.

## Elysia, Web APIs, and Runtime-Specific Facilities

| Capability | Native facility | Use |
|---|---|---|
| HTTP routing and composition | `Elysia`, groups, guards, plugins | Build feature modules without another router or controller framework |
| Boundary validation | Installed Elysia `t` or supported project validator | Reuse existing schemas; validate consumed input and status-specific outputs without duplicating validators |
| Expected responses | `status()` | Return typed status/body unions without exception-driven control flow |
| Error mapping | `.error()` and `onError` | Map known errors and sanitize unexpected failures |
| Request policies | macros, `beforeHandle`, `resolve` | Reuse typed authentication and authorization rules |
| Request context | `derive`, `resolve`, `decorate`, `state` | Add only lifecycle-appropriate values; do not build a service locator |
| Cookies | Elysia cookie API and cookie schemas | Configure signed/secure session cookies when the auth design requires them |
| WebSockets | Elysia WebSocket routes | Add realtime transport without a second HTTP framework |
| Password hashing on Bun | `Bun.password` | Bun deployments only; use the existing supported implementation on another adapter |
| IDs and crypto | Web Crypto and `crypto.randomUUID()` | Generate secure random values; do not invent token algorithms |
| Outbound HTTP | `fetch`, supported timeout/cancellation API | Bound calls; add retries only for an explicit safe/idempotent requirement, never automatic paid-side-effect replay |
| Lifecycle tests | `app.handle(Request)` with the project's runner | `bun:test` is Bun-only; host-adapter integration still needs its own checks |
| Basic request logs | lifecycle hooks plus JSON stdout | Emit structured completion/error events before adding a logger package |
| Security headers | Elysia response headers or trusted ingress | Set the exact CSP/HSTS/frame/referrer/permissions policy the deployment needs |

## Official Packages

| Package | Purpose | Install when |
|---|---|---|
| `@elysia/bearer` | Extract an RFC 6750 Bearer token | Bearer parsing is repeated; it does **not** validate identity or permission |
| `@elysia/cors` | Configure Cross-Origin Resource Sharing | A real browser client calls from another origin; restrict allowed origins |
| `@elysia/cron` | Run scheduled jobs inside an Elysia process | In-process scheduling fits the deployment and duplicate execution across replicas is prevented |
| `@elysia/eden` | Eden Treaty/Fetch end-to-end typed client | A TypeScript client can consume the exported server type with matching Elysia versions |
| `@elysia/apollo` | Integrate Apollo Server GraphQL | The API contract requires Apollo GraphQL; also follow its `graphql`/Apollo peer requirements |
| `@elysia/graphql-yoga` | Integrate GraphQL Yoga | The API contract requires Yoga rather than REST or Apollo |
| `@elysia/html` | Return HTML/JSX with appropriate handling | The Elysia service renders HTML; keep untrusted output XSS-safe |
| `@elysia/jwt` | Sign and verify JSON Web Tokens | JWT is selected with explicit issuer, audience, expiry, algorithm, and key lifecycle |
| `@elysia/openapi` | Generate OpenAPI JSON and interactive documentation | A named consumer needs a published contract; protect internal operations in production |
| `@elysia/opentelemetry` | Instrument Elysia lifecycle and export OpenTelemetry spans | A collector, sampling policy, and concrete trace investigation use-case exist |
| `@elysia/server-timing` | Emit lifecycle timing through the `Server-Timing` header | Browser-visible diagnostics are useful; review exposure before production |
| `@elysia/static` | Serve files and directories | The application, rather than a CDN/object store, must serve bounded static assets |

The official catalog also documents Prisma, Better Auth, React Email, TanStack Start, and deployment integrations. A guide does not make the integrated product an Elysia-owned dependency. Runtime adapters are a separate choice: current Node docs use `@elysia/node`; current Worker docs describe an experimental built-in Cloudflare adapter. Check installed compatibility and runtime restrictions rather than assuming every listed plugin works everywhere.

## Authentication Decision

Choose the credential model before the package:

| Requirement | Default direction |
|---|---|
| First-party browser login with revocation/logout | Reuse the selected mature auth/session implementation; opaque secure-cookie sessions are one option when server-side storage is already appropriate |
| Service/mobile credentials | Reuse the required credential protocol; JWT only when selected with explicit validation and key lifecycle |
| Existing identity provider/OIDC | Use its maintained SDK or mature integration after verifying issuer, audience, JWKS, scopes, and failure behavior |

Authentication proves principal identity. Authorization separately checks the action and resource. Protect login/reset endpoints with shared ingress/store throttling, return generic credential failures, rotate or revoke durable credentials, and never log secrets or complete tokens.

## Logging and Observability

The consulted catalog has no dedicated **official** request-logging package; community loggers are separate. If request logging is required, reuse the project's existing logger, or use correctly scoped lifecycle hooks and the deployment's log sink:

- accept or generate a request ID at a trusted boundary;
- record method, normalized route, status, duration, and safe identifiers;
- log one completion event and one unexpected-error event at most;
- redact credentials, cookies, secrets, database URLs, and sensitive bodies;
- let the hosting platform collect stdout.

Use `@elysia/opentelemetry` for traces, not as a substitute for an operational log contract. Use `@elysia/server-timing` for timing headers, not durable logging.

Add an external logger such as Pino only when transports, serializers, advanced redaction, child logger context, or measured logging throughput justify it. Prefer direct library integration or a thin local Elysia plugin over an unmaintained wrapper.

## Security Fallback Matrix

| Need not covered by an official plugin | First choice | External dependency gate |
|---|---|---|
| HTTP security headers | Exact Elysia hook or ingress policy | Consider a maintained helper only for complex, centrally managed policy generation |
| Distributed rate limiting | CDN/WAF/API gateway or shared datastore | Reject per-process memory for horizontally scaled enforcement |
| CSRF for cookie authentication | Preserve a deliberate strategy combining appropriate SameSite, trusted Origin checks, and tokens where required | SameSite or CORS alone is not universal CSRF protection; verify client/credential semantics before selecting a library |
| Server-side sessions | Reuse selected auth/session infrastructure and secure cookies | Do not add a database/session store without an actual requirement; prefer mature integrations for account/recovery/provider flows |
| Structured request logging | Lifecycle hooks and JSON stdout | Add Pino or another mature logger for demonstrated operational requirements |
| Durable background jobs | Reuse existing deployment/job facilities; consider a DB outbox only for required atomic delivery | A queue/outbox is not a default CRUD layer; define idempotency and duplicate handling first |
| ORM and migrations | Project-selected persistence tool | Prisma is external but justified when selected; read `prisma-postgres.md` |

For every external dependency, record the missing capability, why native/platform facilities are insufficient, maintainer and release health, peer/runtime compatibility, transitive/security cost, state model across replicas, and failure/removal plan.

## Production Consistency

Across projects, keep these invariants even when optional libraries differ:

- repository-native boundaries and preserved Elysia method chaining;
- startup configuration validation and an app export without listening side effects;
- explicit input and status-specific response contracts, including hook failures;
- authentication separate from authorization with tested plugin scope;
- bounded bodies, queries, pagination, timeouts, and only explicitly justified retries;
- stable sanitized failures, preserving intended status and required headers;
- safe correlation/logging when required, without adding telemetry as unrelated scope;
- deployment-appropriate shared enforcement for cross-replica policies;
- focused lifecycle and relevant adapter checks when authorized, reporting only checks actually run.

Do not deploy, push, create paid resources, apply/reset a database, or issue paid/provider side effects as part of dependency selection or verification.

## Official Sources

- [Plugin overview](https://elysiajs.com/plugins/overview)
- [Bearer](https://elysiajs.com/plugins/bearer)
- [CORS](https://elysiajs.com/plugins/cors)
- [Cron](https://elysiajs.com/plugins/cron)
- [Eden](https://elysiajs.com/eden/installation)
- [GraphQL Apollo](https://elysiajs.com/plugins/graphql-apollo)
- [GraphQL Yoga](https://elysiajs.com/plugins/graphql-yoga)
- [HTML](https://elysiajs.com/plugins/html)
- [JWT](https://elysiajs.com/plugins/jwt)
- [OpenAPI](https://elysiajs.com/plugins/openapi)
- [OpenTelemetry](https://elysiajs.com/patterns/opentelemetry)
- [Server Timing](https://elysiajs.com/plugins/server-timing)
- [Static](https://elysiajs.com/plugins/static)
- [Node adapter](https://elysiajs.com/integrations/node)
- [Cloudflare adapter and limitations](https://elysiajs.com/integrations/cloudflare-worker)

Selection and security guidance is original synthesis. Catalog membership is neither an adoption claim nor a substitute for installed-version/API, licensing, and runtime review.
