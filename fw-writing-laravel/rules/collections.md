# Collection Decisions

## Explicit callbacks or higher-order syntax

Both forms are valid. Use a higher-order message only when method return values, keys, side effects and early stopping match the original callback. each() stops when its callback returns false. A callback that invokes a method without returning its result continues; each->method() propagates the result and may stop. Do not replace one solely for brevity.

## Bounded iteration

cursor() hydrates incrementally but does not eager-load relationships; driver buffering can still use memory. lazy()/chunked reads support bounded batches and eager loading. Choose from relation access, dataset size, ordering and connection behavior.

For mutation that changes query membership, prefer stable-key iteration such as lazyById()/chunkById() where appropriate. Do not mutate the pagination key; inspect custom keys, joins, aliasing and concurrent writes. Stable-key iteration is not a snapshot or guarantee of exactly-once processing.

## Bulk queries

Both bounded whereIn()/whereKey() queries and an Eloquent collection's toQuery() can fit. In the Laravel 12.x source baseline, toQuery() rejects empty/mixed-model collections and uses newModelQuery(), without global scopes; it does not verify connection homogeneity. Preserve owner/tenant/soft-delete filters, connection and authorization explicitly. Check the installed implementation before substitution.

Define empty-input behavior, model/connection homogeneity and parameter limits. Bulk updates/deletes bypass per-model save/delete events; use the existing lifecycle when those effects are required.

Use a custom collection only for actual reusable behavior. CollectedBy or newCollection() are alternatives depending on installed versions and project conventions, not reasons to introduce a collection class.

Sources: Laravel 12.x [collections](https://laravel.com/docs/12.x/collections), [Eloquent collections](https://laravel.com/docs/12.x/eloquent-collections) and the [toQuery() implementation](https://github.com/laravel/framework/blob/12.x/src/Illuminate/Database/Eloquent/Collection.php). Source branches can change; inspect the locked/installed implementation for security-sensitive substitutions.
