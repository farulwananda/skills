# Elysia Backend Patterns

Official lifecycle, plugin, validation, error, and runtime documentation consulted on 2026-10-15. This is independent synthesis; current docs are not proof of an installed release's API. Inspect the lockfile, exported types, and matching release docs before changing `status`, hooks/macros, schemas, plugin namespaces, or adapters.

## Contents

- Feature boundaries
- Contracts and validation
- Lifecycle and context
- Dependency selection
- Errors, security, and operations
- Testing and Eden
- Common mistakes

## Feature Boundaries

Follow the existing architecture. A feature-based Elysia instance is useful when it owns a cohesive HTTP contract or cross-cutting plugin; a small service need not become a modular monolith. Preserve method chaining because each call carries new type information.

**Partial integration example:** this module requires the repository's real `ProjectModel` wire schemas and bounded, authorization-appropriate query implementation. It is not a standalone runnable file.

```ts
// src/modules/projects/index.ts
import { Elysia, status } from 'elysia';
import { ProjectModel } from './model';
import { findPublishedProjectBySlug } from './queries';

export const projectsModule = new Elysia({
  name: 'modules.projects',
  prefix: '/projects',
}).get(
  '/:slug',
  async ({ params }) => {
    const project = await findPublishedProjectBySlug(params.slug);

    if (!project) {
      return status(404, {
        code: 'PROJECT_NOT_FOUND',
        message: 'Project not found',
      });
    }

    return project;
  },
  {
    params: ProjectModel.params,
    response: {
      200: ProjectModel.detail,
      404: ProjectModel.notFound,
    },
  },
);
```

Keep HTTP extraction, cookies, headers, and status selection in the Elysia module. Pass only required values to request-independent functions. Never pass the entire `Context` into a controller or persistence function.

Add a service when it owns business decisions, coordinates multiple operations, is reused, or defines an atomic transaction. Do not add a service that only forwards arguments. Do not add a repository interface for a single Prisma implementation; introduce a boundary when a real alternate implementation, domain isolation, or testing seam earns it.

A focused `queries.ts` may hold a reused or non-trivial read without creating a repository abstraction. Keep a one-off bounded query beside its route when that is clearer.

## Contracts and Validation

Use the project's established supported validator. Elysia `t` is a suitable default when none exists because it provides runtime validation and inferred types; Standard Schema support and response/OpenAPI capabilities vary by Elysia release and validator. Do not duplicate an established schema stack merely to use `t`, or assume all validators expose identical capabilities.

Validate every external boundary that is used:

- `params`, `query`, `body`, `headers`, and cookies;
- formats, lengths, ranges, enumerations, and bounded page sizes;
- response bodies for each success and expected failure status, including auth/hook short-circuits;
- uploads with explicit size limits and installed file/content validation; never trust only a claimed MIME header.

Register reusable schemas with `.model()` when that makes route contracts clearer. Do not expose database-generated models as wire contracts: API nullability, field visibility, serialization, and compatibility have different owners.

Use the installed typed status helper (`status(code, body)` in current docs) for expected not-found, conflict, credential, or permission branches. Keep status selection in the HTTP boundary; inner functions return data/domain outcomes or throw deliberate domain errors. Returning a plain `Error` is not a reliable status contract. Unexpected exceptions must remain failures, not success-shaped fallback data.

Treat response-schema failure as a server contract bug, not bad client input. Serialize public fields explicitly; response validation is not a substitute for selecting allowed database fields. Raw `Response`, streams, and error-hook returns may not follow the same validation/type-inference path as a normal handler in every version. Document and test these boundaries rather than claiming they are all covered by a route's success schema.

## Lifecycle and Context

Choose hooks by the first stage that has the required information:

| Need | Mechanism |
|---|---|
| Parse a custom body format | `onParse` |
| Normalize before validation | `transform` |
| Derive data before validation | `derive` |
| Add validated request identity/data | `resolve` or a macro |
| Authorize before handler | `beforeHandle` or a route macro |
| Transform a known handler result | `afterHandle`, before final response mapping |
| Map final output to a Web `Response` | `mapResponse`, only when default serialization is insufficient |
| Convert/log failures | `onError` |
| Emit post-response telemetry | `onAfterResponse` |

Register hooks before the routes/plugins they must affect and verify the composed result. In current docs, local hooks apply to the instance and its descendants, `scoped` also reaches the immediate parent, and `global` reaches consuming ancestors. This does not mean a hook inside an auth plugin protects a consuming sibling route. Use the installed `{ as: ... }` or `.as(...)` form only when that scope is intended; explicit per-feature `.use(auth)` plus a route macro/guard often makes policy ownership clearer. `onRequest` runs before route-specific context exists; do not rely on validated params or resolved identity there.

Use context extensions narrowly:

- `state` is mutable application state; avoid authoritative per-request or business state there.
- `decorate` is for stable shared capabilities, not a service locator.
- `derive` runs before validation.
- `resolve` runs after validation and is appropriate for typed authenticated identity.

Named plugins with a suitable `name`/`seed` can deduplicate hooks; names are not authorization or isolation boundaries. Authentication and resource/action authorization are separate checks.

For authentication, first define the client, credential transport, expiry, logout/revocation, and authorization model. Use secure `HttpOnly` cookies for browser sessions when appropriate. Use `@elysia/bearer` only to extract RFC 6750 bearer credentials and `@elysia/jwt` only when JWT is the selected token format; neither choice replaces authorization. Resolve the current principal into typed request context and check resource/action permission at the route boundary.

## Dependency Selection

Read [official-ecosystem.md](official-ecosystem.md) before choosing a plugin or external substitute. Check current official docs instead of copying old package names.

Add an official plugin only when its behavior is required:

- same-origin deployments do not need CORS;
- an internal service without contract consumers may not need OpenAPI UI;
- JWT is not a default authentication architecture;
- OpenTelemetry is useful when traces have a collector and multiple meaningful spans;
- Eden is valuable for a TypeScript first-party consumer, but it requires matching Elysia versions and resolvable server types.

For capabilities without an official plugin, first inspect whether Web APIs, the selected runtime, platform ingress, or a small local function meets the requirement. Before adding a community package, check maintenance, ownership, compatible peers, adapter/runtime support, transitive dependencies, security history, storage model, and failure behavior. Do not install merely to investigate.

## Errors, Security, and Operations

Use the existing safe error contract; introduce `{ code, message }` only when needed. Map known failures explicitly, and redact stack traces, SQL, secrets, credentials, and sensitive bodies. A policy hook returning an expected error bypasses `onError`; include its status/body in the route contract.

| Failure | HTTP behavior |
|---|---|
| Malformed request body | 400, sanitized parse failure |
| Request validation | Preserve the repository's documented 422 (Elysia default) or deliberate 400 contract |
| Invalid/missing credentials; insufficient permission | 401; 403 respectively, with transport-appropriate headers/cookies |
| Missing route/resource; known uniqueness conflict | 404; 409 respectively |
| Response validation or unexpected implementation/provider failure | 500 with a generic public body; log safely once |

Do not blanket-map every exception to 500 or every `VALIDATION` code to 422: request and response validation differ. Inspect the installed validation error discriminator (for example `error.type`) before implementing a shared mapper. Preserve intentional HTTP statuses and required headers such as `WWW-Authenticate`; do not trust an arbitrary thrown object's `status` as a public contract.

### Runnable contract module: `elysia-greeting-app`

Complete `app.ts` module for Bun with an installed Elysia release exporting `Elysia` and `t`. It has no database, provider call, installation, listening socket, or deployment side effect. It demonstrates one validated endpoint, not a universal error handler; framework parse/validation/not-found responses retain their installed defaults.

```ts
import { Elysia, t } from 'elysia';

export const app = new Elysia().post(
  '/greetings',
  ({ body }) => ({ message: `Hello, ${body.name}!` }),
  {
    body: t.Object({
      name: t.String({ minLength: 1, maxLength: 80 }),
    }),
    response: {
      200: t.Object({ message: t.String() }),
    },
  },
);

export type App = typeof app;
```

For a common error envelope, add real installed-version error mapping and matching status schemas rather than presenting this success-only module as a full application error contract.

At boundaries:

- validate and authorize independently;
- use exact production origins when CORS is needed;
- bound body sizes, list limits, timeouts, and outbound requests;
- keep services stateless across replicas;
- put durable coordination in PostgreSQL or an explicitly selected shared system;
- use structured stdout logging with request ID, route, status, duration, and safe identifiers;
- add a logging package only when transports, serializers, redaction, or performance requirements exceed a small local logger;
- add distributed rate limiting only at a shared edge/store; per-process memory is not replica-safe.

When request logging is required, use one safe structured completion event with the repository's correlation fields (for example timestamp, request ID, method, normalized route, status, and duration). Do not impose tracing or a new logger on unrelated work. Validate/bound client-provided request IDs and only trust proxy headers from configured ingress. Never log credentials, database URLs, or sensitive bodies.

Do not add Redis, a queue, an event bus, microservices, clustering, or an outbox merely to satisfy the word “scalable”. Add them for measured caching value, durable asynchronous side effects, independent deployment/ownership, or another explicit operational requirement.

Do not add retries as a general production requirement. A timeout or lost response does not prove a provider failed to execute. Paid or non-idempotent actions require an explicit authorized retry/idempotency contract; database retry logic must not replay external side effects.

## Runtime and Adapter Boundaries

- Bun-only APIs such as `Bun.password`, `Bun.file`, or Bun server configuration belong only in a Bun deployment; they are not Elysia-portable facilities.
- Current Node docs use `@elysia/node` and `node()`; use the namespace and API supported by the installed release rather than replacing an older working adapter automatically.
- Current Cloudflare docs describe an experimental `elysia/adapter/cloudflare-worker` adapter, `.compile()` at export, and runtime/compatibility-date restrictions. Check the installed Elysia version and Wrangler config; do not copy a Bun `.listen()`, filesystem/static plugin, or old `aot: false` rule into a Worker.
- `app.handle` in Bun tests validates lifecycle logic, not deploy-host compatibility. Use the repository's existing adapter integration tests for bindings, streaming, files, cookies, and shutdown semantics when relevant. Do not deploy to verify.

## Testing and Eden

Export the composed app without `.listen()` for lifecycle tests. **Partial integration example:** the following test requires the actual projects module, app composition, and isolated database fixture proving that `missing` is absent; it is not a self-contained smoke test for the greeting module above.

```ts
import { describe, expect, it } from 'bun:test';
import { app } from '../src/app';

describe('GET /projects/missing', () => {
  it('returns the documented not-found contract', async () => {
    const response = await app.handle(
      new Request('http://localhost/projects/missing'),
    );

    expect(response.status).toBe(404);
    expect(await response.json()).toEqual({
      code: 'PROJECT_NOT_FOUND',
      message: 'Project not found',
    });
  });
});
```

Use full URLs in `app.handle`. Prefer real route behavior; isolate external effects at intentional boundaries. Cover input validation, response validation failures, expected statuses, auth coverage (including routes that should stay public), uniqueness conflicts, pagination, transaction rollback, and sanitized unexpected errors as applicable. Run only authorized existing checks and disclose what was not exercised; Bun tests alone do not establish another adapter's behavior.

Export `type App = typeof app` from the composed server for Eden Treaty. Use `@elysia/eden` on a first-party TypeScript client only when sharing the server type is viable. Keep server/client Elysia versions aligned and TypeScript strict mode enabled.

## Common Mistakes

| Mistake | Better decision |
|---|---|
| Express controller receives `Context` | Let the Elysia instance own HTTP; pass plain values inward |
| Every feature gets route/service/repository/use-case files | Create only boundaries with real behavior |
| Zod duplicates `t` schemas | Use `t` unless an established schema or capability gap exists |
| Global hook appears ineffective | Check registration order and plugin scope |
| JWT plugin is installed before auth requirements | Choose session/token model and trust boundary first |
| CORS defaults to every origin | Omit it for same origin or allow exact trusted origins |
| Per-process limiter is called scalable | Enforce at shared ingress/store across replicas |
| `listen()` is mixed into app composition | Separate `app.ts` from the process entrypoint |
| OpenAPI/telemetry installed “for production” | Require an actual contract consumer or telemetry backend |
| Package copied from an old example | Verify current official namespace and peer versions |

## Pressure Check

| Rationalization | Decision |
|---|---|
| “It is an official plugin, so adding it is free” | Official support lowers risk; it does not create a requirement |
| “A portfolio probably needs admin auth and OpenAPI” | Product labels do not prove consumers or trust boundaries; require the actual contract |
| “Enterprise-grade means standard repository/use-case layers” | Add boundaries for behavior or alternate implementations, not status signaling |
| “A deadline makes a familiar Express stack safer” | Parallel lifecycle and validation systems increase integration risk under time pressure |
| “Scalable from day one requires Redis, workers, or microservices” | Stateless bounded code can scale horizontally; add infrastructure for measured or durable needs |

Red flags are dependencies with no named consumer, empty forwarding layers, per-process coordination presented as horizontally safe, or architecture justified only by “standard”, “future-proof”, “enterprise”, or “production-ready”. Stop and reduce the design to established contracts before implementing.

## Official Sources

- [Best Practice](https://elysiajs.com/essential/best-practice)
- [Validation](https://elysiajs.com/essential/validation)
- [Lifecycle](https://elysiajs.com/essential/life-cycle)
- [Plugin Overview](https://elysiajs.com/plugins/overview)
- [Unit Testing](https://elysiajs.com/patterns/unit-test)
- [Eden Installation](https://elysiajs.com/eden/installation)
- [Plugin scope](https://elysiajs.com/essential/plugin)
- [Error handling](https://elysiajs.com/patterns/error-handling)
- [Node adapter](https://elysiajs.com/integrations/node)
- [Cloudflare adapter and limits](https://elysiajs.com/integrations/cloudflare-worker)
- [Public Elysia skills](https://github.com/elysiajs/skills): consulted for topic coverage and encapsulation principles only. No upstream implementation/prose copied, no license/adoption claim; official API docs and installed code take precedence.
