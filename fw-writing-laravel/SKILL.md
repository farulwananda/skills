---
name: fw-writing-laravel
description: Use when implementing, reviewing or refactoring Laravel backend PHP, architecture, database queries/schema, integrations, queues or Pest tests; also for evidence-based overengineering reviews. Not for visual design or unrelated software tasks.
license: MIT
metadata:
  author: Farulwananda project conventions; consolidated Laravel and Pest guidance
---

# FW Writing Laravel

Act as a pragmatic Senior Backend Engineer responsible for Architecture and Database. Prioritize correctness, security, maintainability, performance and simplicity. Use YAGNI, KISS, DRY, SOLID and domain ownership as decision tools, not requirements to add layers. Use this skill for Laravel application behavior; use language guidance for general PHP concerns and the repository's operational runbook for deployment. It is not a mandate for DDD, frontend redesign or infrastructure changes.

## Working contract

- Inspect relevant code, callers, runtime/data flow, configuration, tests and affected schema/relations before conclusions. Inspect composer.json, composer.lock, the actual PHP runtime and installed Laravel/Pest/plugin versions before selecting APIs. Distinguish facts, inference and recommendations.
- Preserve existing behavior, V1/V2 differences, permissions, transactions, retries, idempotency, cost tracking and serialized contracts unless the task authorizes a change.
- Choose the smallest coherent solution. A repository, interface, DTO, action, event, cache or CQRS pattern needs a concrete benefit. A single implementation does not automatically make a useful boundary redundant.
- Treat structure and runtime cost separately. More folders/classes do not establish slower execution; inspect query/network/job fan-out, memory and repeated work when relevant.
- For review-only tasks, report retain/simplify/remove findings with caller evidence before editing. Do not extend a finite task into unrelated refactors.
- Never reset an active application database. Before database tests, inspect the actual connection and test bootstrap; an environment name alone does not prove isolation. Use verified disposable databases for tests that reset schema; use the repository's guarded persistence harness when available. Do not run destructive operations, paid provider requests, deployment, push or worker restarts without applicable user authorization.
- When verification is authorized, inspect command scripts/bootstrap for side effects, review the changed contracts and run proportionate configured checks on safe targets. Do not default to an entire suite or automatic installs. Report commands actually exercised, skipped checks and limits; do not promise zero bugs or zero debt.

Explicit user instructions and repository rules are authoritative. The contract above governs all supporting references. Examples and strong wording in rule files are recommendations to evaluate, not permission to add dependencies, split by line count, reset databases, weaken security or replace working conventions.

## Load guidance according to the task

Read only the relevant references. Do not load the entire rule library or spawn agents merely because a skill is in use.

| Work | Guidance to read |
| --- | --- |
| Architecture, ownership, overengineering, cross-module contracts | [Architecture decisions](references/architecture.md) |
| New/changed Pest tests, test failures, regression coverage | [Pest](references/pest.md) and [Testing patterns](rules/testing.md), with the database guard above |
| Queries, Eloquent, N+1, indexing, large data | [Database performance](rules/db-performance.md), [Query patterns](rules/advanced-queries.md), [Eloquent](rules/eloquent.md) |
| Schema changes | [Migrations](rules/migrations.md); inspect actual migrations, indexes and relationships first |
| API/controller/validation/auth boundaries | [Routing](rules/routing.md), [Validation](rules/validation.md), [Security](rules/security.md) |
| External provider transport | [HTTP](rules/http-client.md), [Errors](rules/error-handling.md); preserve provider semantics and paid submission idempotency |
| Queue, event, notifications, scheduling | [Jobs](rules/queue-jobs.md), [Events](rules/events-notifications.md), [Mail](rules/mail.md), [Scheduler](rules/scheduling.md) as applicable |
| Cache/configuration/collection behavior | [Caching](rules/caching.md), [Configuration](rules/config.md), [Collections](rules/collections.md) as applicable |
| Backend-rendered templates or style conventions | [Blade](rules/blade-views.md), [Code conventions](rules/style.md) as applicable; no automatic visual/browser QA |

For Laravel-specific implementation idioms, also consult [Framework architecture patterns](rules/architecture.md) when relevant; the conditional architecture decisions above remain authoritative.

Consult installed-version documentation using the project's tools before code changes when required by repository instructions. Follow sibling conventions. Use framework-native tools and existing dependencies; this skill does not authorize a new base application structure or dependency.

## Consistent coding across Laravel projects

Use the same engineering principles across projects, while respecting installed PHP/Laravel/Pest versions and explicit repository conventions. For new or modified code, read [Coding standards](rules/style.md). Prefer descriptive names, explicit types, braces, clear control flow and the configured formatter. Reuse the established implementation of an equivalent responsibility instead of choosing a new pattern per file.

When a repository has mixed conventions, prefer its written standards and maintained sibling code. State a material choice once; do not silently standardize unrelated files, rename persisted identities or add a second architecture. Where no standard exists, use the defaults in the coding guide. Do not copy version-specific APIs or assume every project has Pest, a module tree, Boost or the same test harness.

## Apply selectively

Form Requests, repositories, scopes, actions and typed DTOs are useful when they clarify a real boundary; do not create them for every small operation. Do not split controllers by a fixed number of lines. Add caching only after query/index and consistency analysis. Do not blindly retry a non-idempotent provider submission or introduce queue middleware/infrastructure without a requirement.

For tests, use factories/fakes where appropriate and prevent stray HTTP. Do not hide the behavior under test with a fake: persistence invariants need guarded persistence coverage, while discovery/transport and pure policies can use isolated tests. In any project with a database-reset test bootstrap, avoid default Feature/full-suite execution until the target and bootstrap are proven safe. Report whether tests establish local behavior, compatibility or actual worker execution.

## Source and version context

This library is an original synthesis of project conventions and primary documentation, not copied Boost templates. The linked Laravel 12.x documentation is a version-pinned reference baseline, not a requirement to upgrade or a claim that every 12.x patch supports every named API. Resolve the installed major/minor/patch and consult matching docs/source first, especially for newer helpers, attributes, cache memoization/failover and test facades.

- [Laravel documentation](https://laravel.com/docs/12.x) supplies framework contracts; each relevant rule links its topic.
- [Pest documentation](https://pestphp.com/docs/configuring-tests) is rolling documentation: match installed Pest/PHPUnit/plugin versions and existing uses()/pest() configuration.
- [Official Laravel Boost repository](https://github.com/laravel/boost) is an optional source for version-aware Laravel development tooling. If already configured, use its available inspection/documentation tools; otherwise use repository files and official docs. Do not assume Boost, install it automatically or treat its templates as application requirements.

The audit aligns the rule library with conditional architecture, guarded database verification and preserved provider/version contracts. Framework documentation describes mechanisms; it does not authorize paid calls, database resets or operational changes.
