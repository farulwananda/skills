# HTTP Entry Points

Use route model binding when it fits the actual lookup and route contract. Explicit findOrFail()/scoped queries are also valid. Preserve connection, route keys, owner/tenant filters, soft-delete policy and missing-record behavior. Binding and validation do not replace authorization.

Use scoped nested bindings when parent-child identity must be enforced and the actual relationship supports it; verify inferred/custom relation names. Test unauthorized and mismatched parent/child requests.

Resource controllers suit conventional CRUD endpoints. Explicit routes suit other workflows; do not rename existing public URLs/actions or split controllers merely to resemble resource conventions. Check actual API prefix and registration for the installed project.

Keep controllers focused on transport, authorization/validation and response. Delegate a cohesive workflow when this clarifies responsibility. A short bounded operation can stay in the controller under repository conventions. No fixed line limit and no mandatory action/service per method.

Inline validation and Form Requests are both valid. Use a Form Request when its reusable/substantial boundary helps; otherwise use the validated return value of inline validation. Apply authorization once at the correct boundary without assuming it exists elsewhere.

Preserve status codes, response shape, pagination, redirects and named routes. When verification is authorized, run relevant route/permission regression checks on isolated targets when changing entry points.

Sources: Laravel 12.x [routing](https://laravel.com/docs/12.x/routing), [controllers](https://laravel.com/docs/12.x/controllers) and [authorization](https://laravel.com/docs/12.x/authorization). Match installed registration/binding APIs and preserve existing V1/V2 contracts.
