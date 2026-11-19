# Validation & Forms Best Practices

## Use Form Request Classes

Use Form Requests for substantial/reusable validation and authorization boundaries. Inline validation can be sufficient for a small operation under repository conventions; avoid adding a class just to forward a few rules.

Partial controller-method excerpt: assumes an injected Illuminate\Http\Request, an existing authorized endpoint and the feature's workflow. It validates data only, not authorization or persistence:

```php
$data = $request->validate([
    'title' => ['required', 'string', 'max:255'],
    'body' => ['required', 'string'],
]);
```

Partial controller-method excerpt: assumes $request is the endpoint's existing authorized Form Request:

```php
$data = $request->validated();
```

Both are valid. Apply authorization and the feature's existing workflow before persistence; do not create a Form Request solely to replace a small inline rule set.

## Array vs. String Notation for Rules

Array syntax is more readable and composes cleanly with `Rule::` objects. Prefer it in new code, but check existing Form Requests first and match whatever notation the project already uses. Partial rules-array entries below require Illuminate\Validation\Rule for the array form and the existing users table; they are alternatives, not a standalone script:

```php
// Preferred for new code
'email' => ['required', 'email', Rule::unique('users')],

// Follow existing convention if the project uses string notation
'email' => 'required|email|unique:users',
```

Scope uniqueness/exists checks according to the actual domain and connection; existence does not grant access. Enforce concurrent uniqueness with the appropriate database constraint as well. For updates, never pass user-controlled input to unique()->ignore(); use the authorized model or trusted system-generated identity.

## Validate Payload Shape, Including Nested Keys

Use the validated result of inline validation or FormRequest::validated()/safe() for the intended fields; do not mass-assign raw request data. These APIs do not replace authorization or complete nested-shape validation.

An array rule on a parent alone can return additional nested keys. For a closed payload, allowlist permitted keys and validate each field's type/constraints. Partial rules-array excerpt to place inside an existing validator or rules() method:

```php
$rules = [
    'settings' => ['required', 'array:label'],
    'settings.label' => ['required', 'string', 'max:100'],
];
```

With these rules, settings.is_admin is rejected rather than silently passed to persistence. Adding array:label without validating settings.label would restrict keys but would not validate the label's contents. FormRequest::safe() is not an independent sanitizer for loosely defined arrays.

For intentionally open provider metadata, validate the envelope and required fields according to its actual contract; do not discard unknown raw fields by applying a closed allowlist indiscriminately. Keep that metadata separate from writable business attributes. Preserve null/optional/list semantics and check installed-version validation behavior.

## Conditional required fields still need type validation

Partial rules-array entry in an existing Form Request rules() method; import Illuminate\Validation\Rule and validate account_type separately. Require company_name for business accounts while also validating a supplied value for other accounts:

```php
'company_name' => [
    Rule::requiredIf($this->input('account_type') === 'business'),
    'nullable',
    'string',
    'max:255',
],
```

## Guard Additional Validation

An after() callback still runs when earlier validation fails. Before using a field, check its validation errors and known type/shape. Query only within the authorized owner/tenant scope; a valid or existing ID is not authorization. Keep database queries bounded and avoid probing inaccessible records.

Partial Form Request method: assumes the normal rules already require start_at/end_at as dates, imports listed below, and installed support for FormRequest::after(). This cross-field check adds no database query:

```php
public function after(): array
{
    return [
        function (Validator $validator): void {
            if ($validator->errors()->has('start_at')
                || $validator->errors()->has('end_at')) {
                return;
            }

            $data = $validator->getData();
            if (! isset($data['start_at'], $data['end_at'])
                || ! is_string($data['start_at'])
                || ! is_string($data['end_at'])) {
                return;
            }

            if (CarbonImmutable::parse($data['end_at'])->lessThan(
                CarbonImmutable::parse($data['start_at']),
            )) {
                $validator->errors()->add('end_at', 'End date must not precede start date.');
            }
        },
    ];
}
```

Import Illuminate\Validation\Validator and Carbon\CarbonImmutable, use date formats/timezones consistent with the normal rules, and localize the message following repository conventions. Prefer a built-in rule such as after_or_equal:start_at when it expresses the actual requirement; this example illustrates guarding a custom callback, not a reason to create one.

A stock check during request validation is advisory. Enforce concurrent stock/financial invariants again within the actual transactional workflow. Preserve working withValidator() conventions unless there is a concrete reason to change.

Sources: Laravel 12.x [validation](https://laravel.com/docs/12.x/validation), especially [array keys](https://laravel.com/docs/12.x/validation#validating-arrays), [conditional required rules](https://laravel.com/docs/12.x/validation#rule-required-if) and [Form Request callbacks](https://laravel.com/docs/12.x/validation#performing-additional-validation-on-form-requests). Confirm installed APIs; the six PHP fences are partial contexts, not runnable application implementations.
