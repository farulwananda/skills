# Queue and Job Decisions

## Decide whether a queue is needed

Use direct synchronous execution for bounded validation, queries and calculations when the caller needs the result immediately and the operation fits the request budget. Existing queue usage is not a reason to queue an unrelated small operation.

Consider a job when the result can be delivered later and work is slow/variable, needs independent retries or must survive the request lifecycle. Check HTTP latency/timeouts, provider cost, volume and failure recovery before deciding. A network call alone does not require a queue; queueing critical work also requires reliable dispatch and idempotency.

Before introducing or removing a queue, compare response/status contracts, ordering, transaction timing, polling/broadcast consumers, timeout behavior and worker capacity. Preserve existing queued payloads and recovery semantics. Queue/worker/serialization overhead is a cost to justify, not a performance improvement to assume.

Inspect the configured connection: the sync driver executes immediately and is not durable background execution. Serialized Eloquent models are reloaded, not immutable snapshots; constrained loaded relations may be reloaded without those constraints. Reapply required tenant/owner/query constraints in the worker and preserve payload compatibility with already queued V1/V2 jobs.

## Visibility and timeouts

Set worker/job timeout below the queue visibility window (retry_after on relevant drivers; visibility timeout on SQS). Check actual driver, job timeout, worker options, shutdown grace and external I/O timeouts. Overlapping delivery remains possible; jobs need appropriate idempotency.

## Retry safely

Choose bounded backoff from transient failure and rate-limit semantics. A timeout after submitting paid work may mean accepted, not rejected; reconcile or reuse the existing idempotency/receipt mechanism before resubmission.

Laravel retryUntil() takes precedence over attempt limits in the installed worker. It does not universally require tries=0. Inspect installed-version behavior and maxExceptions/release/timeout semantics before changing existing limits.

## Deduplication and failure handling

Use ShouldBeUnique only when the actual workflow should suppress duplicate dispatch. It does not guarantee exactly-once execution and does not apply to jobs inside batches. Shared cache, lock identity and expiry matter. ShouldBeUniqueUntilProcessing releases its lock at processing start; use it only if concurrent follow-up work is safe.

Implement failed() when there is required terminal recovery or notification. Existing centralized handling can suffice; avoid duplicate refunds, alerts or leaking raw exception messages. failed() can receive a fresh job instance, so mutated handle state is not a reliable recovery snapshot.

## Middleware and batching

RateLimited or WithoutOverlapping can be appropriate for a demonstrated rate/concurrency limit. Middleware release can consume attempts; preserve retry and lock semantics. Do not add middleware merely because an API exists.

Bus::batch() groups jobs and completion/failure callbacks. It is not an atomic transaction: completed jobs and external effects are not rolled back when another job fails. Model partial completion, replay and compensation explicitly when needed; preserve existing workflows.

Database transaction callbacks may be retried after a deadlock. Do not put a paid provider submission inside a retryable transaction closure unless its established idempotency/reconciliation contract makes repetition safe. An after-commit dispatch avoids reading uncommitted state but does not by itself guarantee atomic delivery across database and queue/provider boundaries.

Horizon is optional Redis queue infrastructure, not required for multiple queues. Prefer existing worker/monitoring setup unless requirements justify a dependency and operational change.

Sources: Laravel 12.x [queues](https://laravel.com/docs/12.x/queues), including [serialized relationships](https://laravel.com/docs/12.x/queues#handling-relationships), [unique jobs](https://laravel.com/docs/12.x/queues#unique-jobs), [time-based attempts](https://laravel.com/docs/12.x/queues#time-based-attempts) and [transactions](https://laravel.com/docs/12.x/queues#jobs-and-database-transactions). Inspect installed worker/driver code and configuration before applying optional mechanisms.
