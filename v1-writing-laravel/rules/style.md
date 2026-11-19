# Consistent Laravel Coding Standards

## Precedence and defaults

Follow explicit user/repository rules, configured formatter/static analysis and maintained sibling conventions first. Use these defaults when the project has no standard. Apply them to the authorized change; do not mass-format or rename unrelated code merely to harmonize style.

- Classes/enums use PascalCase; methods, PHP variables and properties use descriptive camelCase. Models normally use singular nouns, tables/config/database columns snake_case following Laravel conventions. Keep existing schema, route, queue and persisted names stable.
- Include parameter/property/return types when the installed PHP version supports them and the contract is known. Use nullable/union types deliberately; use PHPDoc for array shapes, generics and non-obvious contracts. Do not invent types that conceal unknown provider data.
- Use braces, guard clauses and explicit readable flow. Avoid clever one-liners and deep nesting. Cohesive methods need not meet an arbitrary line limit.
- Prefer constructor property promotion where it fits the existing class and PHP version. Preserve framework lifecycle contracts; method injection is appropriate for job handle dependencies and other runtime boundaries.
- Match repository choices for strict_types, date mutability, casts()/casts property, validation notation and imports. Do not introduce alternate styles per file or change their semantics just for consistency.
- Use established services/queries/components for an equivalent responsibility. A new service, repository or DTO needs a responsibility, not merely a naming convention.
- Explain reasons/invariants with concise comments; do not narrate obvious code. Prefer types and descriptive names over redundant comments.

## Helpers without behavior changes

Use native PHP or Laravel helpers according to clarity, required behavior and established conventions. Neither is universally better. Human text may need UTF-8 helpers; binary/signature protocols need byte-accurate operations.

Do not replace input() with a magic property, first()->attribute with value(), explicit ordering with latest(), or null handling with Arr::get() merely for brevity. These forms can have different route fallback, missing/null, accessor/cast or ordering semantics. Check caller expectations first.

Keep substantial reusable JS/CSS in existing assets/components where appropriate. Small Blade/Alpine directives and existing report renderers can remain. Use safe escaping/serialization at the HTML/JS boundary; do not introduce a dependency just to move code.

## Formatting and version compatibility

When verification is authorized, run the repository's configured formatter and focused checks for changed code after inspecting script side effects. Laravel Pint is appropriate when already installed/configured. Do not prescribe one CLI flag or add tooling across all projects; use the project's supported commands.

Check installed versions before choosing newer helpers, attributes or syntax. Maintain a coherent style within the target project without rewriting legacy contracts to match another project's structure.

Sources: [PHP type declarations](https://www.php.net/manual/en/language.types.declarations.php), Laravel 12.x [contribution style](https://laravel.com/docs/12.x/contributions#coding-style) and [Pint](https://laravel.com/docs/12.x/pint). Framework contribution style is context, not a reason to override application conventions; installed PHP and formatter configuration govern supported syntax.
