# External HTTP Boundaries

Set explicit request and connection timeouts appropriate to the provider and workflow; keep existing configuration and timeout contracts unless evidence requires a change.

Check transport status and decoded provider success separately. Laravel does not throw on ordinary 4xx/5xx without explicit handling. Validate required shape/types and handle incomplete, malformed and rate-limited responses without exposing secrets or raw internal errors.

Retry only eligible transient failures with bounded backoff. Retrying a non-idempotent POST after a connection timeout can duplicate a charge or accepted run. Use the existing provider idempotency key, submission receipt and reconciliation flow; do not assume a lost response means no side effect. Keep retries at a clear layer to avoid multiplying nested retries.

Do not add retry() as default client setup. If the workflow needs it, select eligible failures explicitly, respect provider delay/budget limits and inspect the installed signature. In the 12.x baseline, throw: false does not suppress exhausted connection failures. An accepted asynchronous operation still needs its existing receipt/status workflow rather than treating any 2xx as completed work.

Http::pool() can reduce latency for independent calls, but raises concurrent provider load. Use it only with suitable limits, cost budget, response mapping and failure handling. Sequential requests can be the correct choice.

Reuse an existing client or configuration; introduce a macro only if actual repeated setup warrants it. Do not create forwarding-only HTTP layers.

For isolated tests, use Http::fake() and Http::preventStrayRequests() where supported. Assert exact contract-relevant payload/header behavior and failure cases. These mechanisms only cover Laravel's HTTP client; inspect and isolate direct SDK/cURL or other transports separately. A fake cannot prove provider acceptance or extraction quality. Explicitly authorized live diagnostics are a separate workflow and must not leak tokens or trigger paid runs without authorization.

Sources: Laravel 12.x [HTTP client](https://laravel.com/docs/12.x/http-client), [retry behavior](https://laravel.com/docs/12.x/http-client#retries) and [stray-request prevention](https://laravel.com/docs/12.x/http-client#preventing-stray-requests). Follow installed APIs and the actual provider's idempotency/status specification.
