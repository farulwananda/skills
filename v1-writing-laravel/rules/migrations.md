# Schema and Data Changes

Inspect existing migrations, constraints, relationships, data cardinality and installed database capabilities first. Generate migrations with the project's Artisan tooling and --no-interaction. Do not modify deployed migration history; use a focused new change.

Choose foreign-key deletion behavior from lifecycle/retention requirements: restrict/no-action for references that must survive, cascade only for children that should actually disappear, set-null only for valid nullable orphan relationships. These are alternatives, not universal defaults. Soft delete does not invoke database ON DELETE actions.

Check existing data against new constraints, connection boundaries and column types before adding keys. Select indexes from actual queries/EXPLAIN and existing index prefixes; account for write cost and automatically created indexes.

When changing a column, preserve all required existing attributes according to the installed framework/database behavior. Inspect nullability, defaults, length, unsigned/type and index implications rather than assuming unspecified modifiers survive.

A model default may be useful when unsaved instances need a value; a database default remains authoritative for other writers. Mirror defaults only with a clear requirement and keep the definitions consistent.

Separate large/risky backfills from schema changes when locking, partial failure and deployment ordering justify it. Small required seed data can be cohesive; do not introduce extra migrations solely to follow a diagram. Use bounded idempotent backfills with explicit recovery when needed.

Provide down() when reversal is genuinely safe. A rollback that drops populated data is destructive, not automatic recovery. Use a forward fix when appropriate; do not pretend irreversible changes can be safely reversed.

Respect the entrypoint's database guard: no reset/drop/truncate/destructive execution without applicable authorization. Test schema changes only against verified isolated targets. Describe deployment order, compatibility, data risk and rollback limits before activation.

Sources: Laravel 12.x [migrations](https://laravel.com/docs/12.x/migrations) and [database transactions](https://laravel.com/docs/12.x/database#database-transactions). Match the installed schema builder and database engine/version; DDL locking, transactionality and rollback support are not universal.
