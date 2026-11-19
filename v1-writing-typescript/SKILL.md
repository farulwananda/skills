---
name: fw-writing-typescript
description: Use when writing, reviewing, or refactoring TypeScript types, utilities, runtime boundaries, or compiler configuration. Prefer sound narrowing and repository-native conventions; defer framework and deployment contracts to their matching skills.
license: MIT
metadata:
  author: Farul Wananda TypeScript conventions
---

# FW Writing TypeScript

## Scope and repository checks

Use for language and runtime-boundary decisions, not as an application architecture template. Framework behavior belongs to the React, Astro, Express, Elysia, or Cloudflare skill.

Before editing, inspect the installed TypeScript version, package manager/lockfile, `package.json` module type, inherited `tsconfig`, runtime, nearby types and verification scripts. Follow the existing validation/error conventions. Do not install Zod, introduce branded types, or change compiler/module settings merely because this skill is loaded.

## Runtime validation and narrowing

- Treat external JSON, storage and network input as `unknown`; validate every property promised by a predicate. An assertion (`as User`) and `satisfies` do not validate runtime data.
- Prefer inference for local values and explicit contracts at public boundaries. Avoid `any` as an error escape; isolate unavoidable third-party `any` at a checked boundary.
- Reuse an installed schema validator for complex boundaries and infer types from its schemas. A small, accurate guard is enough for a small contract.
- Handle null, arrays, missing properties and wrong nested types. Do not claim a string merely because its key exists.

Self-contained guard; uses TypeScript 4.9+ unlisted-property narrowing:

```typescript
export function isUserWebhook(data: unknown): data is { user: { id: string } } {
  if (typeof data !== 'object' || data === null || Array.isArray(data)) {
    return false;
  }
  if (!('user' in data)) return false;
  const user = data.user;
  return (
    typeof user === 'object' &&
    user !== null &&
    !Array.isArray(user) &&
    'id' in user &&
    typeof user.id === 'string'
  );
}
```

This checks structure, not ownership, authorization, UUID format, or non-empty IDs. Add those only when the actual contract requires them. Older compilers may need a checked record helper instead of unlisted-property narrowing.

## Model only meaningful distinctions

- Use a discriminated union when states have different valid fields. Exhaustive `switch` branches can use `never` to reveal a missing case.
- Optional properties are appropriate for genuinely optional data; their number is not an architecture threshold.
- Branded IDs are useful if accidental interchange is a demonstrated risk. A cast-based constructor is not runtime validation; validate at the real input boundary.
- Use the repository's exceptions or result unions consistently. A new `Result` abstraction is not required for every function.
- Prefer `const`; use `readonly` to communicate ownership where helpful. It is compile-time and shallow, not a runtime freeze. Do not copy arrays/objects just to simulate immutability when ownership permits mutation.
- Avoid non-null assertions without an established invariant. Optional chaining is not a replacement for handling a required value being absent.

## Compiler and module decisions

Do not replace a framework-generated or inherited configuration with a universal baseline.

| Execution model | Decision |
| --- | --- |
| Node executes emitted JavaScript | Match Node version and ESM/CJS package format; modern `NodeNext` resolution models Node's file-extension and package-export rules. |
| Vite/Astro or another bundler resolves imports | Keep the framework's configuration; `bundler` resolution requires TypeScript 5.0+ and a compatible module setting. |
| Bun, Workers, or test runner | Match actual runtime/tooling and generated types; do not assume every Node or DOM API exists. |

Prefer `strict` for new code. Tightening existing settings such as `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, or `verbatimModuleSyntax` is a separately scoped migration: inspect affected callers first. `paths` aliases do not by themselves rewrite emitted imports or teach the runtime how to resolve them.

## Verification

Run the repository's focused typecheck, then execute the changed path in its real runtime. For predicates, exercise valid data and null, numeric/null IDs, missing nested objects, and arrays; compilation alone cannot establish a guard's honesty. For module changes, run the emitted application or build output, not only the editor resolver.

Use consumer-visible boundary/error cases, not tests that pin source text, type aliases, or copies of an implementation. Report which runtime and TypeScript version were exercised; do not claim runtime safety from types alone.

## Sources and provenance

Original project guidance, revised 2026-10-15; linked sources are references, not vendored skill text. Match their API guidance to installed versions:

- [TypeScript narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html), including property narrowing introduced in 4.9.
- [Module reference](https://www.typescriptlang.org/docs/handbook/modules/reference.html) and [TSConfig reference](https://www.typescriptlang.org/tsconfig/).
- [mkosir TypeScript style guide](https://github.com/mkosir/typescript-style-guide): community perspective on inference and consistency, not a mandatory compiler profile.
