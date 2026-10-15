# Testing Best Practices

## Choose Database Isolation Before Refresh Traits

`RefreshDatabase` can run `migrate:fresh`. `LazilyRefreshDatabase` defers the refresh until a query or transaction triggers it; it does not make the target safe or prove that migration will be skipped. Use either only with a verified disposable database and safe bootstrap. For a prepared schema, use the repository's guarded persistence harness and transaction strategy when appropriate. Never use refresh traits against the active application database.

## Match Assertions to the Contract

Use assertModelExists() for existence of a known model. Use assertDatabaseHas() for relevant persisted values, with the correct connection/table. Neither is universally better; existence alone does not prove a state update, amount, owner scope or financial invariant.

Use exact response/status and state assertions when the contract requires them. Do not weaken assertions merely to use a shorter helper.

## Choose Factory Setup by Intent

Reuse a named factory state when it exists and describes the same domain state. Direct attribute overrides are valid for a specific boundary case or when a named state adds no useful meaning. Do not create a state solely to avoid an explicit override, and do not assume every project provides User::factory()->unverified().

Sequences are useful for intentionally varying records; recycle() is useful when related models should share an existing instance. Preserve factory callbacks, relationships and required model events.

## Test Exception Reporting Deliberately

Use Exceptions::fake() when testing exception reporting and the installed framework supports it. withoutExceptionHandling() serves a different purpose: exposing thrown exceptions for debugging. Neither is a universal replacement for the other; preserve the established equivalent on older versions.

## Preserve Required Factory Events

If factory setup depends on model events (for example to generate IDs), faking all events before setup can break it. Fake after required setup or fake only the intended event types; follow the test purpose. Do not fake a listener whose behavior the test must verify.

Partial setup ordering when the factory requires its events: create the fixture first, then fake the specific event whose dispatch is under test, then execute and assert the behavior. Faking before setup is valid when those setup events are intentionally irrelevant; do not adopt one order mechanically.

## Guard External Effects

Fakes assert the boundary they intercept, not actual worker execution, delivery or provider success. Inspect direct SDKs, HTTP, mail, queues, subprocesses and secondary connections; a transaction on the default connection does not isolate all of them. Use existing guarded harnesses and disposable resources, not a new destructive global bootstrap.

Sources: Laravel 12.x [database testing](https://laravel.com/docs/12.x/database-testing), [event fakes](https://laravel.com/docs/12.x/mocking#event-fake) and [exception tests](https://laravel.com/docs/12.x/http-tests#testing-exceptions); [Pest configuration](https://pestphp.com/docs/configuring-tests) is rolling documentation. Resolve the installed versions and test bootstrap before selecting commands.
