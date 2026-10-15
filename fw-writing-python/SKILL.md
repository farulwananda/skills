---
name: fw-writing-python
description: Use when writing, reviewing, or refactoring Python applications, scrapers, pipelines, CLIs, workers, or libraries. Follow the repository's Python version and tooling; emphasize explicit contracts, bounded concurrency, honest missing data, and resource cleanup.
license: MIT
metadata:
  author: Farul Wananda Python conventions; modern software engineering
---

# FW Writing Python

## Scope and repository context

Use this skill for Python language and library code. For HTTP routing, dependency injection, framework lifecycle, or deployment, use the matching framework skill as well. Do not impose web-service architecture on a script or library.

Before editing:
- Inspect `pyproject.toml`, lockfiles, interpreter constraints, CI, nearby code, and existing tests. Confirm the runtime and installed dependency versions in the project's environment; a system interpreter is not evidence of the deployed version.
- Preserve the supported Python range. Built-in generics require 3.9+, `X | None` requires 3.10+, `TaskGroup`, `asyncio.timeout`, and `Self` require 3.11+, and `class Name[T]` syntax requires 3.12+. Do not raise the minimum version just to modernize syntax.
- Reuse the current package manager, test runner, formatter, type checker, and dependencies. Do not install Ruff, uv, Pydantic, browser tooling, retry libraries, or async-test plugins automatically.
- Read the actual input/output and failure contracts before choosing concurrency, retries, data models, or extra layers.

## Language and data contracts

- Annotate public boundaries and non-obvious helpers; prefer precise types over `Any`. Use `collections.abc` interfaces for consumers and concrete containers when mutation or ownership matters. Keep compatibility typing where the supported interpreter needs it.
- Use simple functions and existing modules first. Introduce classes, protocols, services, or repositories only for a real state, substitution, or domain boundary.
- Dataclasses are useful for internal structured records. Choose `slots`, immutability, and keyword-only construction deliberately; slots affect dynamic attributes, inheritance, and some serialization patterns. Use the existing validation library at external boundaries when needed, not for every internal value.
- Distinguish missing, empty, invalid, and unavailable data. A missing article title is `None` or an explicit parse error according to the contract—not `"Untitled Article"`, invented body text, or a successful empty article.
- Avoid mutable defaults, wildcard imports, hidden mutation, and coercions that disguise invalid data. Keep logging's deferred formatting when useful; f-strings are not a universal replacement.

### Honest article metadata

Runnable snippet `python-article-contract` (Python 3.10+, stdlib only). This accepts a JSON metadata object, not HTML, and does not validate whether its `source` string is a safe fetch URL. Validate fetch destinations separately.

```python
import json
from dataclasses import dataclass


class ArticleFormatError(ValueError):
    """Article metadata does not match the input contract."""


@dataclass(frozen=True, slots=True, kw_only=True)
class Article:
    source: str
    title: str | None


def parse_article(raw: str) -> Article:
    try:
        data = json.loads(raw)
    except json.JSONDecodeError as exc:
        raise ArticleFormatError("Article metadata is not valid JSON") from exc

    if not isinstance(data, dict):
        raise ArticleFormatError("Article metadata must be an object")
    source = data.get("source")
    if not isinstance(source, str) or not source.strip():
        raise ArticleFormatError("Article source must be a non-empty string")
    title = data.get("title")
    if title is not None and not isinstance(title, str):
        raise ArticleFormatError("Article title must be a string or null")
    return Article(source=source.strip(), title=(title.strip() or None) if title else None)
```

## Exceptions, ownership, and cleanup

- Catch specific exceptions where the code can recover or translate a boundary. Preserve causes with `raise DomainError(...) from exc`; let unexpected programming errors propagate. Do not expose credentials, full upstream responses, or private input in errors.
- Use `with` / `async with`, or `try/finally` when a resource has no context manager. Define who creates, owns, shares, and closes each file, client, connection, browser, and task.
- Use `contextlib.ExitStack` / `AsyncExitStack` when acquisition is conditional or involves several resources. Cleanup must cover failures during later acquisition, normal completion, errors, and cancellation.
- Do not swallow `asyncio.CancelledError`; clean up and propagate it. Async deadlines cannot interrupt blocking code or guarantee that a remote side effect did not complete.

## Concurrency and deadlines

- Keep sequential work sequential unless concurrency has a reason. Async is useful for awaitable I/O; calling blocking code inside `async def` still blocks the event loop. A normal helper is not automatically offloaded.
- For unavoidable blocking I/O, use an existing executor boundary or `asyncio.to_thread` where the runtime supports it. Cancellation does not stop the underlying thread; keep the library's own timeout and bound submitted work. Threads do not generally make pure-Python CPU work parallel under the GIL.
- Bound active operations **and task creation**. A semaphore around millions of pre-created tasks does not bound memory. For streams, use a bounded queue and a fixed number of workers; for small finite batches, chunking is sufficient.
- On Python 3.11+, use `TaskGroup` when sibling cancellation and grouped failures match the contract. It is not a mechanical replacement for `gather`: `gather` has different failure behavior. Decide whether results are all-or-nothing or explicitly partial.

Runnable snippet `python-bounded-map` (Python 3.11+, stdlib only). It preserves order, limits tasks to one chunk, and applies a per-operation deadline. A failure cancels siblings and propagates an exception group; no partial result is returned. Output still occupies O(n) memory, and each chunk waits for its slowest member. This is not a streaming or rate-limiting API.

```python
import asyncio
from collections.abc import Awaitable, Callable, Iterable
from itertools import islice
from typing import TypeVar

T = TypeVar("T")
R = TypeVar("R")


async def bounded_map(
    items: Iterable[T],
    operation: Callable[[T], Awaitable[R]],
    *,
    concurrency: int = 8,
    timeout: float = 10.0,
) -> list[R]:
    if concurrency < 1 or timeout <= 0:
        raise ValueError("Concurrency and timeout must be positive")

    async def run_one(item: T) -> R:
        async with asyncio.timeout(timeout):
            return await operation(item)

    iterator = iter(items)
    results: list[R] = []
    while batch := list(islice(iterator, concurrency)):
        async with asyncio.TaskGroup() as group:
            tasks = [group.create_task(run_one(item)) for item in batch]
        results.extend(task.result() for task in tasks)
    return results
```

For older supported runtimes, reuse the project's concurrency implementation with explicit cancellation and awaiting of owned tasks; do not silently upgrade Python.

## Scrapers, browser automation, and workers

- Set explicit connection/read/pool timeouts supported by the client and an overall deadline when required. Bound response size, pages, fan-out, and navigation work according to the task.
- Follow authorization, robots/site policy, provider limits, and the repository's identity/header configuration. Do not mandate realistic browser impersonation or user-agent rotation, or bypass access controls.
- Retries are optional policy, not boilerplate. Add them only for documented retryable failures with safe operation semantics, a bounded attempt/time budget, backoff, and provider guidance such as `Retry-After`. A 429/5xx is not by itself proof a submission is safe to repeat.
- For paid job submissions or other side effects, a timeout means outcome may be unknown. Do not automatically resubmit. Use an existing provider-supported idempotency key or status lookup/reconciliation contract; never claim a local lock guarantees cross-worker or crash-safe deduplication.
- Reuse a client/browser for its intended owner lifetime. Isolate browser contexts per request/job or tenant when cookies or credentials differ; close pages and contexts in `finally`, then close the owner browser/client at shutdown.
- With BeautifulSoup/Scrapy/Playwright already present, check selector results and distinguish extraction failure from genuinely missing optional fields. Record an explicit failure or partial result instead of fabricated fallback content.

## Verification and delivery

Use the repository's existing checks when execution is authorized; do not rewrite tool configuration or add plugins as a prerequisite. Select tests for valid and malformed inputs, missing/empty values, cause chaining, concurrency limits, timeout/cancellation cleanup, and side-effect deduplication where relevant. Keep tests offline with explicit fixtures/test doubles; never make paid calls in a smoke test.

Report changed behavior, supported versions, checks actually run, and unresolved limits. If an interpreter, dependency, network credential, representative fixture, or real provider contract is absent, state that limitation rather than inventing successful behavior.

## Sources and selective follow-up

Authored synthesis of the primary sources below; consult the documentation for the **target interpreter version**, not unconditionally the newest release:
- [Python typing](https://docs.python.org/3/library/typing.html) and [dataclasses](https://docs.python.org/3/library/dataclasses.html): compatibility and record semantics.
- [Python 3.11 asyncio tasks](https://docs.python.org/3.11/library/asyncio-task.html): task groups, cancellation, timeouts, and thread offloading.
- [Python exception handling and chaining](https://docs.python.org/3.11/tutorial/errors.html): precise recovery and preserved causes.
- [HTTPX clients](https://www.python-httpx.org/advanced/clients/): pooling, configuration, and lifecycle when HTTPX is already used.
