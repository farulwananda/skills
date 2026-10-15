# Error Handling

Use the project's established exception rendering/reporting strategy and installed-version registration. Exception methods, a legacy Handler and bootstrap configuration can be valid; do not force a whole-project conversion. Co-location and centralized handling may coexist for different responsibilities.

Distinguish expected domain rejection, provider failure and unexpected faults. Preserve API status/payload and UI behavior. Do not suppress exceptions to make tests pass or expose raw provider errors, secrets or stack traces to users.

Do not report an expected exception only if that policy is intentional; ShouldntReport and configured exclusions are alternatives. Throttling or duplicate-report protection can reduce noisy logs when justified, but must preserve actionable failure evidence.

Render JSON according to the actual API contract and content negotiation; do not assume every API lives under api/* or change all routes automatically.

Add structured context containing safe identifiers and relevant state. Do not log raw credentials, personal payloads or arbitrary exception messages indiscriminately. Test reporting/rendering separately when necessary; an exception fake and disabled exception handling serve different purposes.

Sources: Laravel 12.x [error handling](https://laravel.com/docs/12.x/errors) and [HTTP exception tests](https://laravel.com/docs/12.x/http-tests#testing-exceptions). Registration locations and test facades vary by version; inspect the application's installed bootstrap/handler before changing them.
