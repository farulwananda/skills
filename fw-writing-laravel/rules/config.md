# Configuration and Secrets

Read runtime settings through config() and existing configuration keys. env() belongs in configuration/bootstrap paths appropriate to the project; cached configuration changes runtime env() behavior. Preserve existing keys/defaults and do not duplicate configuration.

Keep plaintext secrets out of version control, logs and shared documentation. Follow the existing deployment secret mechanism. Laravel environment encryption or a platform secret store can be appropriate, but neither is required merely to use this skill. Confirm key distribution, filesystem permissions and recovery before changing the mechanism. Encryption keys must not be stored beside encrypted files in the same repository.

Use the application's environment API for runtime environment checks. Do not rewrite operational environment names or deployment behavior merely for style.

Use existing constants/enums for shared meaningful states. A one-off literal does not automatically need a new constant/enum. Preserve serialized values and database meanings.

Follow existing localization conventions for user-facing text. Do not introduce language files or a new translation system solely to relocate a message.

Source: Laravel 12.x [configuration](https://laravel.com/docs/12.x/configuration). Check the installed framework and existing deployment/runbook before changing encryption or configuration caches; do not print resolved secret values while inspecting configuration.
