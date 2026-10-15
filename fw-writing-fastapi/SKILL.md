---
name: fw-writing-fastapi
description: Use when building, reviewing, or refactoring FastAPI APIs. Inspect installed versions and existing conventions; apply typed HTTP contracts, correct async boundaries, explicit resource and transaction ownership, isolated tests, and public response schemas without mandatory layers.
license: MIT
metadata:
  author: Farul Wananda backend conventions; modern FastAPI engineering
---

# FW Writing FastAPI

## Scope and repository context

Use this skill for FastAPI routing, schemas, dependencies, lifecycle, and API tests. Use `fw-writing-python` for language/concurrency details. Do not use this skill alone to choose deployment infrastructure, redesign a non-FastAPI application, or introduce an ORM.

Before editing, inspect the application entrypoint, routers, dependencies, tests, `pyproject.toml`, lockfile, and CI. Establish the supported Python version and the **installed** FastAPI, Starlette, Pydantic, HTTPX, ORM, and async-test plugin versions in the project environment. Do not assume current online documentation matches an older lockfile.

- Prefer existing conventions and dependencies. `Annotated` is available in Python 3.9+; use the repository's compatibility approach where necessary. Pydantic v2 examples do not justify silently migrating a v1 application.
- Do not require service/repository layers, async database drivers, settings packages, caches, retries, or new tooling. A small endpoint may call a query or domain function directly. Extract reusable policy or workflows when complexity warrants it.
- Use `APIRouter` for real domain/version boundaries, not a mandatory folder hierarchy. Preserve existing authentication, authorization, tenant isolation, rate limits, and idempotency guards.

## Execution boundaries

| Code being invoked | Execution behavior |
|---|---|
| FastAPI-managed `async def` endpoint/dependency | Runs on the event loop; awaited I/O must be async-compatible |
| FastAPI-managed ordinary `def` endpoint/dependency | FastAPI/Starlette offloads it to a threadpool |
| Ordinary sync helper called directly inside `async def` | Runs inline on the event-loop thread; **not automatically offloaded** |

Choose `async def` for awaitable I/O or small non-blocking work; choose `def` for a fully synchronous blocking stack. For mixed stacks, explicitly offload only the blocking boundary with the project's existing threadpool helper or `asyncio.to_thread` on supported asyncio runtimes. Bound submitted work; cancellation does not terminate the underlying thread or reverse a remote submission. CPU-heavy Python work needs an appropriate existing process/job boundary or measured alternative, not a claim that threads make it parallel.

Do not share a mutable database session across concurrent tasks. Verify thread safety before offloading a client/session. Async functions containing `requests`, blocking browser APIs, sync ORM calls, file reads, or `time.sleep` still block unless explicitly offloaded.

## HTTP and data contracts

- Type path/query/header values directly; use Pydantic body models for structured validation. Not every scalar parameter needs its own model.
- Keep input, internal, persistence, and **public output** shapes separate when they differ. A return model or `response_model` validates, documents, serializes, and filters output; never leak credentials, tokens, internal job payloads, or ORM-only fields.
- For Pydantic v2, use `model_validate`, `model_dump`, and `ConfigDict(from_attributes=True)` only when needed. For older installations, follow their supported API. Distinguish omitted PATCH fields from explicit `null` with the installed model's unset-field support.
- Use `Decimal` or existing integer-minor-unit contracts for exact money values; do not introduce float rounding into paid workflows.
- Missing optional scraped fields may be `None`; malformed/failed upstream data is not a successful fabricated article. Define whether partial results are permitted and expose their status honestly.
- Use `HTTPException` for expected HTTP-boundary errors; reuse domain handlers for shared domain policy. Map only known failures. Do not turn every exception into a 400 or conceal programming errors as success. Keep sensitive details out of public error messages and preserve internal causes.
- Preserve authorization before reads/writes and server-controlled fields; validation and public output filtering do not replace access control.

## Resource ownership and a bounded outbound call

Shared connection pools and async clients belong to application lifespan, normally one per worker process. Request/job-owned resources belong in dependencies or context managers with deterministic cleanup. Do not create clients at import time, open a browser per selector, or leave pages running after cancellation.

Runnable snippet `fastapi-lifespan-app`: save as `example_app.py`. Requires Python 3.11+, FastAPI with lifespan/Annotated support, Pydantic v2, HTTPX, and an asyncio ASGI runtime. It is a complete, narrow example for checking a **fixed public URL**, not a generic proxy, trend24 adapter, or paid-submission implementation. Importing it makes no network calls; calling its route without the test override below does.

```python
import asyncio
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from typing import Annotated

import httpx
from fastapi import Depends, FastAPI, HTTPException, Request
from pydantic import BaseModel

DOCS_URL = "https://docs.python.org/3/"


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    async with httpx.AsyncClient(
        timeout=httpx.Timeout(5.0, connect=2.0),
        limits=httpx.Limits(max_connections=4, max_keepalive_connections=4),
        follow_redirects=False,
    ) as client:
        app.state.http_client = client
        app.state.outbound_slots = asyncio.Semaphore(4)
        try:
            yield
        finally:
            del app.state.http_client
            del app.state.outbound_slots


async def get_http_client(request: Request) -> httpx.AsyncClient:
    return request.app.state.http_client


HttpClientDep = Annotated[httpx.AsyncClient, Depends(get_http_client)]


class UpstreamStatus(BaseModel):
    url: str
    status_code: int


def create_app() -> FastAPI:
    app = FastAPI(lifespan=lifespan)

    @app.get("/python-docs/status")
    async def python_docs_status(
        request: Request, client: HttpClientDep
    ) -> UpstreamStatus:
        try:
            # Includes time waiting for a slot; HTTPX also limits individual I/O phases.
            async with asyncio.timeout(6.0):
                async with request.app.state.outbound_slots:
                    async with client.stream("GET", DOCS_URL) as response:
                        response.raise_for_status()
                        status_code = response.status_code
        except (TimeoutError, httpx.TimeoutException) as exc:
            raise HTTPException(504, "Upstream deadline exceeded") from exc
        except httpx.HTTPError as exc:
            raise HTTPException(502, "Upstream request failed") from exc
        return UpstreamStatus(url=DOCS_URL, status_code=status_code)

    return app


app = create_app()
```

The stream context closes without buffering the document body. The semaphore limits active outbound calls per process, not total incoming requests or a fleet-wide quota; choose queue/admission limits from the actual deployment. For user-supplied fetch URLs, retain destination validation and SSRF/redirect protections rather than copying this fixed-URL example unchanged.

If Playwright is already required, lifespan may own the browser process. Each job/request or tenant owns an isolated browser context and pages; close them in `finally`, then close the browser at shutdown. Keep navigation timeouts and bounded page/context concurrency. Do not share authenticated cookies across users.

### trend24 and paid jobs

Preserve existing trend24-endpoint timeout, lifecycle, fan-out, and duplicate-submission protections. Inspect its actual provider request/response contract rather than inventing fields or fallback articles.

- Set provider-aware timeouts and an overall operation budget; limit both active work and task creation. Reuse owned clients/browser resources.
- A timeout or disconnect after submission may leave the provider job running. Do not automatically retry paid submissions or schedule the same operation in a background task after already awaiting it.
- Reuse provider-supported idempotency keys and the existing durable job identity/status reconciliation where available. A semaphore, an in-memory set, or response cache cannot guarantee deduplication across workers, crashes, or uncertain remote outcomes.
- If the provider offers no safe repeat/status contract, report the uncertain outcome and missing prerequisite. Do not fabricate success or introduce new infrastructure without a real requirement.

## Dependencies and transaction ownership

Prefer `Annotated[T, Depends(provider)]` where supported and reuse aliases for repeated dependencies. Keep providers overrideable. Dependencies that acquire request resources should yield with guaranteed close/rollback cleanup; never suppress a caught exception after `yield`.

Choose **one** transaction owner per unit of work and keep it explicit:
- The session dependency can own session lifetime while a route/domain operation owns `begin`/commit/rollback. Or use the repository's existing transaction-owning dependency if its semantics are intentional. Do not commit in both.
- Complete commit and any required flush/refresh before returning success. Do not rely on a commit after dependency `yield` if cleanup may occur after the response was sent; yield timing/scopes vary by FastAPI version and streaming behavior.
- Helpers may flush when needed but must not silently commit an outer transaction. Roll back on failure, close sessions, and map known constraint conflicts at the appropriate boundary.
- An external paid submission and a local SQL commit are not atomic together. Preserve the real workflow's idempotency/reconciliation guarantees instead of claiming a database transaction rolls back the remote effect.

Settings follow the existing project approach. If `pydantic-settings` v2 is already used, settings can be a typed dependency; environment files are local configuration, not a secret store. Do not add cached settings by default. If caching already exists, clear it when tests change the environment.

## Isolated tests and override cleanup

Runnable snippet `fastapi-offline-test`: save as `test_example_app.py` next to `example_app.py`. Requires the app snippet, pytest, and compatible FastAPI/Starlette/HTTPX `TestClient` dependencies. It checks upstream failure/timeout mapping, private-detail exclusion, lifespan cleanup, and override restoration. MockTransport is an explicit test double, not application fallback data; no network request is made.

```python
from collections.abc import AsyncIterator

import httpx
import pytest
from fastapi.testclient import TestClient

from example_app import create_app, get_http_client


@pytest.mark.parametrize(
    ("upstream_status", "expected_status"),
    [(503, 502), (None, 504)],
)
def test_upstream_failure_is_safe_and_resources_close(
    upstream_status: int | None, expected_status: int
) -> None:
    app = create_app()

    def handle(request: httpx.Request) -> httpx.Response:
        if upstream_status is None:
            raise httpx.ReadTimeout("private-provider-detail", request=request)
        return httpx.Response(upstream_status, text="private-provider-detail")

    async def override_client() -> AsyncIterator[httpx.AsyncClient]:
        async with httpx.AsyncClient(transport=httpx.MockTransport(handle)) as client:
            yield client

    previous = app.dependency_overrides.copy()
    app.dependency_overrides[get_http_client] = override_client
    try:
        with TestClient(app) as client:
            owned_client = app.state.http_client
            response = client.get("/python-docs/status")
            assert response.status_code == expected_status
            assert "private-provider-detail" not in response.text
        assert owned_client.is_closed
        assert not hasattr(app.state, "http_client")
    finally:
        app.dependency_overrides.clear()
        app.dependency_overrides.update(previous)
```

For real endpoints, select tests for validation, authorization/tenant isolation, public-field exclusion, not-found/conflict behavior, upstream timeout/failure, cancellation cleanup, and duplicate paid submissions. Extend the mock handler to raise the matching HTTPX exception; do not call a paid provider to test its error path.

- Use a fresh application/override scope per test. Restore overrides in `finally` or fixture teardown, including when setup/assertions fail.
- Use `with TestClient(app)` to run lifespan. For async tests with `ASGITransport`, HTTPX alone does **not** trigger lifespan. Reuse the project's lifespan fixture/manager and async plugin; do not add a new plugin automatically. Create loop-bound resources in the loop that uses them.
- Database tests must explicitly target a disposable test database/schema, never development/production by default. Override the DB dependency, isolate parallel workers, and clean up only test-owned data/resources.
- Match the installed ORM's transaction semantics: rollback fixtures require routes to use the test-bound connection/session; routes that commit may need an outer transaction plus supported savepoints or a disposable database. SQLite is not proof of PostgreSQL-specific behavior. Never issue a broad reset against a configured non-test database.

## Verification and sources

When execution is authorized, use existing checks and exercise the affected contract. Report installed/supported versions, checks actually run, and unavailable DB/provider/browser coverage. No automatic installs, deployment, paid calls, or database resets.

These rules are authored synthesis, not a copy of upstream policy. Read references selectively for the relevant feature and installed versions:
- [FastAPI official skill](https://github.com/fastapi/fastapi/blob/master/fastapi/.agents/skills/fastapi/SKILL.md): researched guidance on typed dependencies, public responses, and async boundaries. Its `master` advice changes and may describe APIs absent locally; library/tool preferences are not adoption requirements here.
- [FastAPI async technical details](https://fastapi.tiangolo.com/async/#very-technical-details): managed endpoints/dependencies versus directly called helpers.
- [FastAPI lifespan](https://fastapi.tiangolo.com/advanced/events/) and [dependencies with yield](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/): resource lifecycle and version-sensitive teardown.
- [FastAPI response models](https://fastapi.tiangolo.com/tutorial/response-model/): validation and public-field filtering.
- [Dependency overrides](https://fastapi.tiangolo.com/advanced/testing-dependencies/) and [async tests](https://fastapi.tiangolo.com/advanced/async-tests/): dependency replacement and lifespan caveats.
- [Python 3.11 asyncio tasks](https://docs.python.org/3.11/library/asyncio-task.html) and [HTTPX clients](https://www.python-httpx.org/advanced/clients/): deadlines, cancellation, pooling, and cleanup.
