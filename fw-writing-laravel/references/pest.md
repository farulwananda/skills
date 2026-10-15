# Pest Guidance

Use this reference when the project already uses Pest. Follow installed versions and sibling tests; do not add Pest/plugins or convert PHPUnit merely to apply the skill. Use available version-specific documentation tools, or official docs when the project has no such tool.

## Scope and isolation

Inspect tests/Pest.php, base test classes, extensions, bootstrap and actual database connections before choosing a command. A filename/filter or APP_ENV=testing does not bypass a destructive bootstrap. Run persistence/schema tests only on verified disposable targets; use the guarded harness when available. Pure policies/transport can use isolated tests with appropriate fakes.

When verification is authorized and isolation established, run focused tests first and broaden when failures/new changes or unresolved risk justify it. Use the project's supported command and flags, inspecting Composer scripts for side effects. If scaffolding with Artisan, use --no-interaction and supported --pest/--unit options; avoid duplicating suite directories in the generated name.

## Assertions and fixtures

Match test()/it(), datasets and mocking style to maintained siblings. Reuse factories/states where they fit; direct overrides remain valid. Fixtures must identify synthetic versus captured data and contain no secrets.

Assert the exact contract: assertOk()/assertStatus(200) for exact 200, assertCreated()/assertStatus(201) for exact 201, or assertSuccessful() only when any 2xx is acceptable. Persistence checks must cover required values/owner/rollback behavior, not just existence.

Use Http::fake() and prevent stray requests for isolated transport tests. Do not fake the behavior under test: event discovery, listener execution and financial persistence require corresponding real boundaries or a verified test double at the appropriate layer. Mail/notification/job fakes assert dispatch, not successful delivery or actual worker execution.

PreventStrayRequests covers Laravel's HTTP client, not arbitrary SDKs, raw cURL, SMTP, queue workers or subprocesses. Inspect the actual transport and block/fake its boundary too. A passing fake test cannot establish live provider compatibility, pricing, retries or idempotency.

## Architecture and browser testing

Test repository-agreed dependency rules against actual namespaces/classes. Normal Laravel controller inheritance is valid; do not ban it or introduce Domain folders just to satisfy a sample. Ensure the target classes exist so an empty selection does not make a rule pass vacuously.

Browser/visual verification is optional and follows authorized user/repository scope. Do not run it automatically for backend-only changes. Confirm runner/plugin support, authentication state and isolated data before execution.

Report commands/results and limits honestly. Do not delete coverage or weaken assertions to get a green result; preserve compatibility regressions when they protect real consumers.

## Sources and versions

Primary references: [Pest configuration](https://pestphp.com/docs/configuring-tests), [Pest mocking](https://pestphp.com/docs/mocking), [Laravel database testing](https://laravel.com/docs/12.x/database-testing) and [HTTP test isolation](https://laravel.com/docs/12.x/http-client#preventing-stray-requests). Pest pages are rolling; Laravel links use a 12.x baseline. Verify the installed package APIs and configuration rather than copying current bootstrap examples.
