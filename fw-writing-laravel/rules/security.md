# Backend Security Boundaries

## Input and authorization

Treat external/user input as untrusted. Apply authentication, authorization and owner/tenant scope at the appropriate boundary. Existing middleware, policies, gates and Form Request authorization are valid mechanisms; do not add duplicate checks blindly or assume route binding/exists validation grants access.

Explicitly select writable fields from validated payloads and define mass-assignment protection according to the existing model. Never pass arbitrary request/provider attributes into business models. Validate nested shapes and permitted keys; validated()/safe() does not sanitize loosely specified arrays. Keep open raw provider metadata separate from writable business attributes.

Bind SQL values. Dynamic identifiers need an allowlist. Prevent command/path traversal and validate uploaded content/type/size independently of client filenames. Generate storage names, select public/private storage intentionally and enforce access for private media.

Escape output for its HTML/JS/URL context. Raw HTML requires deliberate trusted sanitization. Preserve CSRF on browser state-changing routes; API/webhook exemptions need the appropriate authenticity controls. Use meaningful auth/API rate limits from abuse and provider capacity requirements.

For server-side URL fetching, inspect scheme, destination/redirect handling and existing SSRF policy; input validation alone does not establish a safe destination. Verify webhook signatures over the required raw body, replay/duplicate behavior and idempotent processing when the provider supports it.

## Secrets and logs

Keep plaintext secrets out of source, version control, output and logs. Use existing config/secret mechanisms. Log safe IDs/state rather than raw credentials, personal data or provider error payloads. Hidden model attributes restrict serialization, not explicit access/logging.

For a new recoverable secret field, an encrypted cast may fit. For a secret only needing verification, appropriate one-way hashing can fit. Choose from actual consumers and the existing security contract.

For existing columns, inspect stored format and every reader/writer before changing to encrypted: plaintext is not automatically encrypted on read. Plan ciphertext column size, key storage/rotation/recovery, rollout compatibility and bounded idempotent conversion without double encryption or data loss. Do not execute a conversion as incidental refactoring.

Encrypted values cannot normally be queried/indexed by plaintext contents. Inspect lookup/uniqueness requirements and installed-version key rotation before changing storage. Test writes, reads, serialization, legacy conversion and failure behavior on isolated targets. Do not introduce new secret infrastructure merely to follow this guide.

Run existing dependency audit tooling when the task/CI policy calls for it; do not add dependencies or weaken controls simply to make implementation easier.

Sources: Laravel 12.x [authorization](https://laravel.com/docs/12.x/authorization), [mass assignment](https://laravel.com/docs/12.x/eloquent#mass-assignment), [encrypted casts](https://laravel.com/docs/12.x/eloquent-mutators#encrypted-casting) and [CSRF protection](https://laravel.com/docs/12.x/csrf). Match the installed version and actual provider signature/security specification; framework helpers alone do not establish tenant isolation or safe SSRF handling.
