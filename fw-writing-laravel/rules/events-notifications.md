# Events and Notifications

Events are appropriate for actual subscribers, broadcasting or a meaningful lifecycle boundary; explicit direct coordination can be simpler when ordering/results matter. Do not add events to replace every service call.

Inspect configured discovery directories and explicit mappings. Module listeners require matching registration; avoid duplicates. Verify cached/uncached manifests when moving classes and follow the authorized deployment runbook. Do not activate caches or restart workers as part of a read-only review.

For effects that must observe committed state, use the existing event/listener/connection after-commit mechanism supported by the installed version. Preserve synchronous listeners intentionally participating in a transaction. afterCommit delays dispatch; it does not guarantee atomic external delivery with database commit.

Queue external notifications when asynchronous execution/retries fit the actual workflow. Preserve intentional synchronous delivery. Notification afterCommit behavior must be checked for queued delivery; setting a queue option does not automatically defer a synchronous notification.

Use dedicated queues only for demonstrated channel workload/priority isolation. Use on-demand routing for recipients without an appropriate model rather than inventing dummy records. Locale preference mechanisms are optional and follow existing localization conventions.

Preserve broadcast channels, payloads, queue names, authorization and serialized identities. Review pending/retry/failed compatibility before deleting old identities.

Sources: Laravel 12.x [events](https://laravel.com/docs/12.x/events) and [notifications](https://laravel.com/docs/12.x/notifications). Discovery, after-commit interfaces/options and queue behavior must match the installed version; no automatic worker/cache activation or real message delivery.
