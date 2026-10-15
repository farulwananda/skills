---
name: fw-writing-cloudflare
description: Use when building, configuring, or reviewing Cloudflare Workers, Pages, static assets, or existing D1, KV, and R2 bindings. Covers version-aware routing, generated binding types, runtime limits, local development, cache isolation, and authorized deployment boundaries.
license: MIT
metadata:
  author: Farul Wananda Cloudflare conventions; serverless edge engineering
---

# FW Writing Cloudflare

## Scope and repository inspection

Use for Cloudflare runtime and hosting concerns, not as a mandate to use Cloudflare, introduce storage, or replace framework architecture. Inspect package.json/lockfile, installed Wrangler and framework adapter versions, scripts, existing Wrangler configuration, generated type files, compatibility date/flags, routes, environment names, migrations, and deployment instructions before changing anything. Keep a working TOML configuration unless a real requirement calls for conversion; JSONC is useful for schema tooling but not mandatory.

Workers run in isolates, not a full Node.js host. Check each dependency's actually used APIs against the installed tooling and configured runtime. `nodejs_compat` does not implement every Node API: some modules are partial implementations or importable stubs whose operations throw or no-op. Whether a flag is needed also depends on the compatibility date; do not add it or advance the date indiscriminately. Current docs describe Node compatibility enabled by default from 2026-08-04, whereas earlier date ranges require explicit opt-in. Preserve and verify the project's compatibility policy rather than copying a new date.

No automatic installs, provisioning, deployments, domain changes, remote migrations, database resets, paid calls, or secret uploads. Determine the exact account/environment and obtain authorization for remote mutations. Commands below are local examples, not permission to execute them.

## Choose one deployment shape

Do not combine these into a universal configuration:

| Shape | Routing/configuration boundary |
| --- | --- |
| Static multi-page site (including prerendered Astro) | Assets only, ordinary HTML handling and real 404s; no Worker `main` or framework adapter needed solely to serve static output. |
| Explicit client-routed SPA | Use `single-page-application` fallback only when the application's client router owns unknown navigation paths. Keep API failures and protected paths out of that fallback. |
| Worker API | Worker `main` and generated binding types; no assets/SPA configuration unless it actually serves a UI. |
| Adapter-generated SSR/full-stack framework | Use the installed adapter/plugin's generated entrypoint, asset directory, config redirection, and deployment command. Do not overwrite these with generic `main: src/index.ts` or assume `dist` is entirely static. |
| Existing Pages deployment | Retain Pages-specific build output and Functions conventions; do not silently migrate it to Workers. Its routing and fallback defaults differ. |

**CF-STATIC-CONFIG** — complete standalone `wrangler.jsonc` for Workers static assets. Requires an existing `dist/` static build; include `dist/404.html` for a custom error page. The example date is a pinned example, not an upgrade instruction:

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "static-site-example",
  "compatibility_date": "2025-04-01",
  "assets": {
    "directory": "./dist",
    "html_handling": "auto-trailing-slash",
    "not_found_handling": "404-page"
  }
}
```

`auto-trailing-slash` follows the built file shape: directory indexes get slash URLs, standalone HTML files get non-slash URLs. Align Astro's canonical/trailing-slash/build format with the host's actual redirects. On Pages, ensure a top-level 404 file when a multi-page site needs real not-found behavior; do not rely on SPA defaults.

For an intentional SPA, change only its asset fallback to `single-page-application` after confirming client routing and deep-link behavior. When mixing Worker APIs and assets, inspect the installed version's routing precedence. Asset-first serving can bypass the Worker; use supported `run_worker_first` path selection for routes that require API/auth execution, and implement their explicit 404s. A protected file must not remain publicly available via a second asset path. SPA navigations and API fetches can have different fallback behavior; test both, not just the homepage.

For Astro SSR specifically, inspect the installed `@astrojs/cloudflare` docs: newer adapters generate configuration through the Cloudflare Vite plugin, while older adapters use different output/runtime access patterns. Keep generated configuration generated. Static Astro needs no SSR adapter; server features do. Adapter image/session bindings can introduce resource usage or automatic provisioning at deployment: review before authorizing it.

## Generated types, not a parallel Env definition

Use the repository's installed Wrangler command through its package manager or existing script, not an executor that may auto-download a newer binary. Wrangler 4's `wrangler types` normally writes `worker-configuration.d.ts`, including binding and runtime types matched to configuration/date/flags. Older Wrangler versions have different runtime-type support; read their command help/documentation. Regenerate after binding/config changes, include the generated file in tsconfig, and never hand-edit it or duplicate binding declarations in a drifting `interface Env`.

Use the environment type/name produced by the installed generator (`Env` in this example); newer tooling also exposes `Cloudflare.Env`. Add only source-owned application augmentations through that version's supported mechanism. Generate for the selected environment where environment bindings differ. Secrets may need explicit supported declarations when not inferred from local secrets/configuration; do not invent production values just to satisfy types.

**CF-API-CONFIG** — complete alternative `wrangler.jsonc`, not merged with CF-STATIC-CONFIG:

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "worker-api-example",
  "main": "src/index.ts",
  "compatibility_date": "2025-04-01"
}
```

**CF-HEALTH-WORKER** — complete `src/index.ts` paired with CF-API-CONFIG and its generated types. Intentionally no database, authentication, caching, observability service, or Node flag:

```typescript
export default {
  fetch(request: Request): Response {
    const url = new URL(request.url);
    if (url.pathname !== '/health') {
      return new Response('Not Found', { status: 404 });
    }
    if (request.method !== 'GET') {
      return new Response('Method Not Allowed', {
        status: 405,
        headers: { Allow: 'GET' },
      });
    }
    return Response.json({ status: 'ok' }, {
      headers: { 'Cache-Control': 'no-store' },
    });
  },
} satisfies ExportedHandler<Env>;
```

## Local development and resource safety

Use the existing local dev script or installed `wrangler dev` with local bindings by default. Inspect `remote: true`, remote binding options, framework dev proxies, service bindings, and provider integrations first: a local dev server does not prove every resource is simulated locally. Use isolated local persistence if needed and avoid sharing state across unrelated test cases. Remote dev can mutate real resources and incur costs; obtain authorization first.

- Keep local credentials in ignored `.dev.vars*` or the project's supported secret mechanism. Avoid committing `.env*` secrets too; retain intentional sanitized example files. Never print secrets in logs, generated documentation, errors, or client bundles.
- Wrangler `vars` is for non-secret configuration. Binding names, D1 database IDs, KV namespace IDs, R2 bucket names, and account identifiers are configuration identifiers, not authentication secrets. They still merit the project's metadata/privacy policy; do not confuse redacting an ID with protecting an API token.
- Bindings grant server-side access; do not expose credentials to the browser. Validate input and enforce authentication, authorization, and tenant scoping in handlers; possession of a binding is not user authorization.
- Add D1/KV/R2 only for a real requirement. Respect KV consistency/expiration behavior and storage-specific semantics. No blanket database, retry layer, cache layer, or queue requirement.
- For D1 use prepared statements and bound values. Dynamic table/column identifiers need an allowlist, not value binding. Apply reviewed migrations locally first (`wrangler d1 migrations apply DB --local`, with the installed CLI and actual binding); remote application requires separate authorization and target checks. Do not use resets as a fix. Preserve transaction/batch semantics and idempotency safeguards; retrying writes is not automatically safe.

**CF-D1-QUERY-FRAGMENT** — partial handler fragment, not a standalone Worker. Requires generated `env.DB`, a migrated `users` table, validated `email`, and prior authorization/tenant scoping if applicable:

```typescript
const user = await env.DB
  .prepare('SELECT id, email FROM users WHERE email = ?')
  .bind(email)
  .first();
```

## Public caching and private isolation

Do not add caching to every response. Establish freshness and isolation requirements first:

- Only mark genuinely public, identical-for-all-callers content `public`. Never put authenticated responses, session cookies, authorization-dependent errors, or tenant data into a URL-only shared cache. Bypass shared cache lookup/write for private routes and use `Cache-Control: private, no-store` where private responses must not persist. Headers alone do not repair an unsafe explicit Cache API write.
- An explicit shared-cache key must represent every response-varying dimension (including tenant/language/query policy where applicable). Do not assume `Vary`, `Authorization`, or a cookie automatically partitions a manually chosen key. Prefer bypassing shared caching for authenticated data over inventing a fragile key.
- Distinguish browser cache headers, Cloudflare CDN behavior, static asset caching, and Workers Cache API. Setting `s-maxage` does not prove that a dynamic Worker response has been cached. Cache API entries are local to the serving data center and are not durable authoritative storage.
- Apply immutable year-long browser caching only to content-addressed/versioned assets, not every font or mutable public filename. `_astro/*` may be fingerprinted; inspect the build output before applying that rule.
- `_headers`/`_redirects` apply to the supported static asset serving path, not arbitrary responses generated by a Worker/Pages Function/SSR handler. Set headers on those responses in code. A `/*.html` rule does not necessarily match extensionless canonical page URLs; check actual public paths. Preserve existing CSP/security headers rather than inventing a blanket policy.

## Promise lifecycle and streaming

Await work whose success is required for the response, especially persistence that the response claims completed. For independent best-effort work, pass an already-created promise to `ctx.waitUntil(promise)`; do not pass an async function or leave an unobserved floating promise. Handle rejection deliberately without exposing secrets; `waitUntil` is neither durable delivery nor a retry guarantee.

For HTTP invocations, current Workers documentation allows up to 30 seconds of post-response/disconnect extension, shared across `waitUntil` calls. While a response body is actively streaming to a connected client, the invocation remains active without `waitUntil`; this does not remove CPU/memory/other runtime limits. Await required results or stream them as produced, preserve backpressure/cancellation, and avoid buffering large bodies or cloning unbounded streams merely for logging/caching. Once a stream has started, a later failure cannot become a fresh HTTP error status. Only introduce a durable queue/workflow when the actual delivery/lifecycle requirement warrants it and the user authorizes infrastructure changes.

## Verification and reporting

When authorized, use existing repository checks plus installed local Wrangler/runtime preview. Generate binding types before checks that depend on them. Exercise health/method/unknown-route responses, static deep links and real 404s, SPA fallback only where intended, API routing precedence, public versus authenticated/tenant cache isolation, secret exclusion, and streamed cancellation/background failures. Local emulation does not prove production routing, limits, remote binding permissions, or cache behavior. Report exact commands run, selected environment/date/tool versions, and unverified remote behavior. Never deploy or access remote data as an implicit smoke test.

## Sources and provenance

Author synthesis informed by primary documentation and selective public references, not bulk-copied third-party skill content. Read only relevant sections and match installed versions:

- Official Cloudflare: [asset routing modes](https://developers.cloudflare.com/workers/static-assets/routing/), [HTML handling](https://developers.cloudflare.com/workers/static-assets/routing/advanced/html-handling/), [Wrangler configuration](https://developers.cloudflare.com/workers/wrangler/configuration/).
- Official Cloudflare: [generated TypeScript types](https://developers.cloudflare.com/workers/languages/typescript/), [Node compatibility and limitations](https://developers.cloudflare.com/workers/runtime-apis/nodejs/), [context and waitUntil](https://developers.cloudflare.com/workers/runtime-apis/context/), [Cache API](https://developers.cloudflare.com/workers/runtime-apis/cache/).
- Official Astro: [Cloudflare adapter](https://docs.astro.build/en/guides/integrations-guide/cloudflare/) for framework-generated SSR boundaries.
- [cloudflare/skills](https://github.com/cloudflare/skills) is Cloudflare's public agent-guidance repository, a researched reference rather than the source of this author's custom policy or a guarantee for every installed version. Do not copy its installation commands or provision services just because they appear in guidance.
