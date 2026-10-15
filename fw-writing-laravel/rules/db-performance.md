# Database Performance

Prioritize query shape, rows scanned/returned, relation loading and network calls before micro-optimization. Inspect actual callers, expected growth, schema/indexes and plans. Do not claim speed improvements from moving files or reducing class count.

- Eager-load relationships actually accessed across result sets to prevent N+1; constrain fields/rows while retaining required primary/foreign/pivot keys. Do not preload unused relations.
- Lazy-loading detection can help in development when supported and compatible with the codebase. Do not enable it globally during unrelated work without understanding affected paths.
- Select fields needed by callers, accessors, serialization and relation matching. Do not remove fields solely to avoid SELECT * when their consumers have not been traced.
- Use bounded pagination/chunks/lazy iteration for large datasets. Choose stable-key iteration when mutations affect membership; preserve ordering and account for driver buffering/concurrent changes.
- Use relation aggregates when consumers need counts/totals instead of the full collection. Preserve filters and avoid multiplying rows through joins.
- Choose indexes from actual filters, joins, sort/range patterns and cardinality. Check existing indexes and EXPLAIN; avoid redundant indexes and consider write/storage cost.
- Prefer improving a poor query or missing justified index before adding cache. Define freshness/invalidation when caching is actually needed.

Keep query retrieval out of presentation templates and preserve owner/tenant/permission scope. Optimize within the feature's existing boundary, without adding a repository just to relocate one simple query.

Source: Laravel 12.x [Eloquent retrieval and relationships](https://laravel.com/docs/12.x/eloquent) and [query builder](https://laravel.com/docs/12.x/queries). Match installed APIs; execution plans, buffering and index behavior depend on the actual database engine/version.
