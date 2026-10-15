---
name: fw-writing-express
description: Use when implementing or reviewing Express HTTP APIs in TypeScript. Covers version-aware request validation, async errors, security boundaries and server lifecycle without requiring a new architecture or dependency.
license: MIT
metadata:
  author: Farul Wananda backend conventions; TypeScript Express standards
---

# FW Writing Express

## Scope and repository checks

Use for Express routing/middleware, HTTP contracts and lifecycle. Use the TypeScript skill for language decisions, not as a reason to reconfigure a working application.

Inspect `package.json`, the lockfile, Express and `@types/express` major versions, Node support, middleware order, proxy/auth configuration, error responses and existing scripts. Match Express 4 versus 5 deliberately. A small handler does not need a controller class, service, repository, schema library or dependency-injection framework.

## Version and validation boundaries

- Express 5 forwards rejected promises returned by handlers to error middleware; Express 4 needs the established promise wrapper or explicit `catch(next)`. Detached callbacks and background promises still need an error owner in both.
- In Express 5, `req.query` is a getter, not a writable property. Do not replace it after parsing. Keep validated data in local variables or typed `res.locals`; do not mutate `req.params` either to simulate a domain object.
- `req.body`, query and path values are untrusted. Reuse the installed validator when useful; infer types from the successful parse. A cast does not validate data.
- Make middleware order explicit: body parsing when needed, authentication/authorization, validation, handlers, not-found handling, then four-argument error middleware. Authentication must still protect routes whose validation succeeds.
- Check Express 5 route syntax and query-parser differences before migrations. Do not copy Express 4 wildcard patterns or assume nested query parsing.

## Runnable Express 5 example

Requires Express 5, matching Express types, Node 18+ and a TypeScript runner/compiler configured by the consuming repository. This app module has no database or external I/O and does not open a port on import. Its arithmetic endpoint demonstrates checked query input without rewriting the request; it is not an application blueprint.

```typescript
import express, { type ErrorRequestHandler } from 'express';

class HttpError extends Error {
  constructor(
    readonly status: 400 | 404,
    message: string,
  ) {
    super(message);
  }
}

export const app = express();
app.disable('x-powered-by');

app.get('/square', (req, res) => {
  const value: unknown = req.query.value;
  if (typeof value !== 'string' || !/^\d{1,6}$/.test(value)) {
    throw new HttpError(400, 'value must contain one to six decimal digits');
  }
  const number = Number(value);
  res.json({ square: number * number });
});

app.use((_req, _res, next) => {
  next(new HttpError(404, 'Route not found'));
});

export const errorHandler: ErrorRequestHandler = (
  error: unknown, _req, res, next,
) => {
  if (res.headersSent) {
    next(error);
    return;
  }
  if (error instanceof HttpError) {
    res.status(error.status).json({ error: error.message });
    return;
  }
  console.error('Unhandled request error', error);
  res.status(500).json({ error: 'Internal server error' });
};

app.use(errorHandler);
```

Only known, client-safe errors expose messages. Map parser/validation/domain errors using the application's established contract; do not expose arbitrary `error.message`, SQL details, request credentials or internal stacks. Preserve a body parser's intended 400/413 status rather than turning every error into 500. After headers are sent, delegate to Express's error path instead of sending a second response. Use the project's redacting logger for unexpected failures; avoid logging raw provider payloads or secrets.

## Security decisions, not middleware cargo cults

- Set body size/content-type limits for accepted payloads. Signature-verified webhooks may need raw bytes before JSON parsing.
- CORS is a browser response policy, not authentication. Same-origin APIs may need no CORS middleware. For cross-origin use, configure the actual allowlist; missing configuration must not silently become `'*'`. Credentials require explicit origins and an intentional cookie/CSRF policy.
- Reuse established security headers (for example Helmet) and tune CSP to the actual client. Add rate limiting only with a threat/traffic requirement and correct shared/proxy behavior.
- Set `trust proxy` to the actual trusted hop/address topology, never blindly to `true`; otherwise IP-based controls may trust attacker input.
- Authorization, parameterized database access and safe output remain necessary even after input validation.

## Resource and shutdown ownership

Keep app construction separate from `listen()` so tests can close every server they start. The bootstrap owns the listener, database pools and HTTP clients.

On shutdown: mark unready, stop accepting requests with `server.close`, allow in-flight work to finish, then close owned pools/clients. Define a finite drain deadline according to deployment requirements and report a nonzero result on failure. Make signal handling idempotent; do not call `process.exit(0)` before asynchronous cleanup finishes. Account for long-lived sockets/upgraded connections using APIs supported by the installed Node version. A callback printing “closed” does not establish that database cleanup occurred.

## Verification

Run configured focused tests/typecheck, then make real requests to an isolated local listener. Exercise valid and invalid input (including repeated/structured query values), 404s, returned async rejection, safe 500 responses and the headers-sent delegation path. Verify middleware and authorization precedence when changed. Use existing Supertest/test tooling if present; it may bind a temporary socket internally and is not proof of deployed end-to-end behavior.

For lifecycle changes, start the actual bootstrap, send its shutdown signal, observe in-flight completion and owned resource closure. Never connect a smoke run to a production database or paid provider. Report precisely which runtime paths were exercised.

## Sources and provenance

Original project guidance, revised 2026-10-15; these are API references, not copied community templates:

- [Express 5 migration](https://expressjs.com/en/guide/migrating-5/): query getter, route syntax and promise rejection changes; Node 18+ baseline.
- [Express error handling](https://expressjs.com/en/guide/error-handling.html).
- [Express production security](https://expressjs.com/en/advanced/best-practice-security.html) and [proxy configuration](https://expressjs.com/en/guide/behind-proxies.html).
- [Node HTTP server lifecycle](https://nodejs.org/api/http.html#serverclosecallback): check installed-version support before using connection-closing helpers.
