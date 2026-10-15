# Laravel Implementation Choices

Follow the [conditional architecture decisions](../references/architecture.md). These are options, not mandatory layers.

- Use an action/service for a cohesive workflow when it improves understanding or testing. A private helper or existing service may be enough; do not extract merely by line count.
- Prefer explicit constructor or method injection for stable dependencies. Container resolution can be appropriate at composition roots, job handle boundaries or genuinely dynamic/scoped resolution. Inspect lifetime before replacing it.
- Contracts are useful at meaningful module/provider boundaries; concrete injection is also valid. Preserve different provider semantics instead of inventing interchangeability.
- Choose deterministic ordering from the feature contract, including tie-breakers when paginating. Do not silently replace business order with latest-first.
- Use transactions and appropriate row/unique constraints for persisted invariants. Cache locks are coordination tools with expiry and shared-store requirements, not substitutes for database correctness.
- Use multibyte helpers for human text; byte-oriented operations remain correct for binary data, signatures and serialized protocols.
- defer(), where supported, is an optional same-process mechanism for noncritical work; it still consumes application process capacity and is not durable. Do not replace durable/retryable jobs or change existing timing without evidence.
- Context can carry diagnostic metadata across queues and logs; do not use unvalidated context for authorization or put secrets in propagated context.
- Concurrency adds processes/connections and load. Use only when independent work, driver support and measured benefit justify it. Preserve transactions, limits and failure semantics.
- Keep framework conventions where they fit; explicit mappings for existing schemas and compatibility are legitimate.

Sources: Laravel 12.x [container](https://laravel.com/docs/12.x/container), [helpers](https://laravel.com/docs/12.x/helpers#deferred-functions), [context](https://laravel.com/docs/12.x/context) and [concurrency](https://laravel.com/docs/12.x/concurrency). Check installed versions and driver/lifetime semantics before using optional APIs.
