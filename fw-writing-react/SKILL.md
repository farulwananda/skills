---
name: fw-writing-react
description: Use when writing, reviewing, or refactoring React and TypeScript UI components in Farul Wananda projects, including accessible primitives, compound APIs, state ownership, effects, data loading, and repository-native styling.
---

# FW Writing React

## Overview

Write cohesive React components: reuse accessible behavior, expose small composable parts, and preserve the target repository's DOM, styling, and state contracts. Base UI, CVA, Tailwind, and slot attributes are optional conventions, not prerequisites.

The existing Selia provenance points to [`55ba90ba64a090653217a2b3afd60537c7b12444`](https://github.com/nauvalazhar/selia/tree/55ba90ba64a090653217a2b3afd60537c7b12444). Treat it as a historical architectural reference, not proof of current upstream parity or a freshly repeated full audit. Local conventions and installed APIs win.

## Scope

Use for React DOM components and their composition/state/data-flow review. Do not use as a substitute for React Native, a router's server APIs, backend design, or deployment instructions. Identify Next.js, TanStack Router/Start, plain Vite, or another host before selecting framework-specific patterns.

## Workflow

1. Inspect nearby components, `package.json` and lockfile versions (`react`, `react-dom`, React types, primitives, router), aliases, CSS tokens, and existing checks.
2. Classify the change as a primitive, a meaningful compound family, or feature composition; preserve the project's placement and export conventions.
3. Search for an existing component before adding state, context, variants, or dependencies. Prefer composition over parallel abstractions.
4. Use semantic HTML for native interactions. For complex behavior, reuse the installed accessible primitive. If it is Base UI, preserve its installed state, event, ref, and `render` contracts; do not install or migrate to it merely to match this skill.
5. Define state ownership and async behavior first. Support the existing controlled/uncontrolled contract, derive render data without effects, and use the host's established loaders/query layer rather than creating request waterfalls.
6. Implement the smallest API. CVA is useful only for repeated visual dimensions when the project uses it; context is useful only for genuinely shared behavior.
7. Use the repository's focused type/tests and proportionate build checks when authorized. Check keyboard/focus, disabled/invalid states, controlled/uncontrolled behavior, ref/event forwarding, and relevant themes. Report checks actually run and any limits; do not install, deploy, push, reset data, or call paid services automatically.

## Component Contract

- Type native wrappers with `React.ComponentPropsWithRef<'tag'>` when exposing a ref; inherit primitive props without duplicating their contract.
- Match the runtime **and types**: React 18 ref forwarding uses `forwardRef`; React 19 can accept `ref` as a prop. Do not silently upgrade a React 18 API or break a library's supported version range.
- Preserve caller events, refs, `aria-*`, `data-*`, form attributes, and children. When adding internal events/refs, compose them using the installed primitive's helpers and cancellation semantics rather than overwriting either side.
- Merge `className` through the existing helper, preserving function-valued primitive classes where supported. Order spreads deliberately; protect fixed semantics without dropping caller extensibility.
- Use stable `data-slot` parts and state selectors only where that is an existing styling contract, not on every incidental node.
- Preserve project tokens and CSS tooling; do not import Selia's palette, Tailwind version, aliases, or export layout wholesale.
- Use the installed polymorphism API (`render`, `asChild`, or another existing convention). Preserve semantics, focusability, disabled behavior, events, and refs across the rendered element.
- Accessibility is behavioral: preserve labels, keyboard support, focus visibility/restoration, error associations, and reduced-motion preferences. Styling alone cannot establish it.

## Decision Reference

Read [references/react-patterns.md](references/react-patterns.md) before implementing a new primitive, compound component, or polymorphic API. It contains the audited structure, code pattern, and common mistakes.

## Done When

The result fits the repository, preserves accessibility and state/composition contracts, and introduces no unnecessary dependencies or framework assumptions. Relevant checks pass if exercised; otherwise state what remains unverified.
