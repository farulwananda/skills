# Eloquent and Query Builder

Choose relation types and casts from the actual schema and data meaning, with supported types and existing conventions. Do not force casts() over the casts property, immutable dates over mutable dates, or newer attributes merely for style.

Use a local scope for a real shared query constraint. Global scopes affect administrative, reporting and background queries too; use them only for deliberate universal policy and preserve existing behavior. whereBelongsTo() and explicit foreign-key filters are alternatives; verify the actual relation.

Handle nullable dates and preserve timezone/format semantics. A stored date-only value is not automatically a timezone-bearing timestamp.

Literal table names are valid in Query Builder, migrations and analytical SQL. DB::table('orders') is appropriate for direct queries; do not create a model/repository/table-name wrapper solely to avoid a literal.

Use Eloquent when existing relations, scopes, casts or lifecycle are required. Query Builder does not apply Eloquent owner/tenant/global scopes, soft-delete filtering or casts. Bulk Eloquent updates/deletes also bypass per-model lifecycle events; inspect the actual operation.

getTable() can help a query follow an existing model's dynamic table mapping. Verify the connection separately; a table name does not select the model's connection. Preserve aliases and schema qualification.

Bind input values. Dynamic table/column identifiers require fixed names or a strict allowlist, not raw user input. Fixed names in historical migrations avoid dependence on evolving models.

Sources: Laravel 12.x [Eloquent](https://laravel.com/docs/12.x/eloquent), [relationships](https://laravel.com/docs/12.x/eloquent-relationships), [casts](https://laravel.com/docs/12.x/eloquent-mutators) and [query builder](https://laravel.com/docs/12.x/queries). Check installed PHP/framework capabilities and existing lifecycle contracts rather than adopting newer attributes/cast syntax automatically.
