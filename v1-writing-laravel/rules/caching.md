# Cache Decisions

Add caching only for an evidenced repeated cost after inspecting queries/indexes. Define freshness, owner/permission scope, keys, invalidation, failure behavior and expected benefit first.

Cache::remember() is useful cache-aside syntax; it does not automatically prevent a cache stampede. Choose locking only if concurrent recomputation requires coordination.

Cache::flexible() serves stale values while refreshing; use only where stale data is acceptable. Do not replace authoritative balances, authorization or transactional state with stale cache. Bound cached data; caching User::all() does not solve an unbounded query.

Cache::memo() reduces repeated store access within a request/job but can retain an earlier value. Check mutation/lifetime requirements. once() also needs a valid memoization lifetime; do not freeze dynamic runtime gates or user-dependent state accidentally.

Tags depend on store support; do not assume file, database or DynamoDB support them. Verify the configured implementation. Group flushing is not a substitute for a complete invalidation strategy.

Cache::add() provides atomic conditional insertion on supported stores. Lock expiry and crashes still matter; use Cache::lock() and ownership semantics when appropriate. Protect persisted business invariants with database constraints/transactions.

Failover stores can improve availability but may split coordination state between processes. Do not enable them blindly for locks, deduplication or idempotency. No new cache infrastructure without a concrete requirement.

Sources: Laravel 12.x [cache](https://laravel.com/docs/12.x/cache), [tags](https://laravel.com/docs/12.x/cache#cache-tags) and [atomic locks](https://laravel.com/docs/12.x/cache#atomic-locks). flexible(), memo() and failover availability depend on the installed framework release; do not infer support from a major-version documentation link alone.
