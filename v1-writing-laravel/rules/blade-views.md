# Backend-Rendered Views

Preserve existing templates, components and V1/V2 behavior. Put database retrieval in the feature's controller/Livewire/query owner rather than initiating unbounded queries in Blade; preserve permissions and relation loading.

Use components when explicit props/slots and reuse improve clarity. Existing includes are valid; do not convert them solely for preference. Shared view data can use an existing view composer when it prevents real duplication, with appropriate owner scope and bounded queries.

Merge attribute bags where consumers should extend classes/attributes. Escape untrusted content appropriately for HTML/JS/URL contexts; only render raw HTML when deliberately trusted/sanitized.

Use pushOnce for actually repeated component assets where the installed version supports it. Aware props and fragments are optional tools for an existing need; do not add frontend dependencies to use them.

Backend-only changes do not automatically require browser/visual verification. Follow the authorized scope and repository checks; preserve rendering behavior when views are changed.

Source: Laravel 12.x [Blade](https://laravel.com/docs/12.x/blade). Confirm installed directive/serialization support and escaping requirements for the actual output context; this guidance does not require frontend tooling.
