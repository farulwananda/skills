# Query Optimization Choices

Inspect existing scopes, query grain, cardinality, indexes and EXPLAIN before proposing optimization. Fewer SQL statements are not automatically faster; consider rows scanned, round trips, memory and write cost. No blanket preference for EXISTS, IN, joins or subqueries.

- A scalar subquery can replace loading an entire relationship for one value. Use indexed correlation and deterministic tie-breaking; preserve null/missing semantics.
- A virtual foreign key plus relationship may hydrate a selected related model efficiently. Use only when real consumers need it; ordinary relationships or bounded eager loads can be simpler.
- Conditional aggregates can combine counts when filters and grain are shared. Keep predicates equivalent and prevent join multiplication. Bind input values in raw expressions.
- An already-loaded parent can be assigned with setRelation() if it truly is the same relation/instance expected by children; do not override a relation with mismatched data.
- whereHas()/EXISTS and IN can have different relation scopes, null and morph semantics. Compare equivalent plans, not syntax alone.
- Secondary ID queries can be useful when bounded/selective; avoid large PHP ID lists and database parameter limits. Account for consistency between separate reads.
- Composite indexes must fit actual equality/range/sort patterns and existing index prefixes. Verify the plan; do not index every filtered column or assume one order works for every database.
- Has-many joins can multiply parent rows. Aggregation, a suitable subquery or a relationship query may preserve the intended grain; do not ban joins when their cardinality is correct.

An optimization must preserve owner/tenant scope, ordering, timezone/date boundaries and results. Report measured changes or label the expected benefit as inference.

Sources: Laravel 12.x [advanced subqueries](https://laravel.com/docs/12.x/eloquent#advanced-subqueries), [relationship queries](https://laravel.com/docs/12.x/eloquent-relationships#querying-relationship-existence) and [raw expressions](https://laravel.com/docs/12.x/queries#raw-expressions). Confirm installed APIs and engine-specific execution plans; these are optimization options, not mandatory query patterns.
