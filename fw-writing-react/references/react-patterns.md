# React Component Patterns

Historical reference: [Selia at `55ba90ba64a090653217a2b3afd60537c7b12444`](https://github.com/nauvalazhar/selia/tree/55ba90ba64a090653217a2b3afd60537c7b12444). The earlier skill recorded this snapshot on 2026-04-12; this review confirmed the commit exists, not every historical component/count claim. Patterns below are an independent synthesis, not copied upstream implementation or a claim of current parity.

Read the relevant section when changing reusable components, state ownership, effects, or loading. Inspect installed React/runtime/types, primitives, CSS, and host-framework versions first.

## Architecture

Selia's recorded separation between reusable primitives and composed product blocks is useful when the target has those boundaries. Kebab-case files, named compound parts, and colocated variants are conventions to reuse where present, not reasons to reorganize a repository.

Use semantic HTML for native behavior and an existing accessible primitive for complex interactions. Base UI and CVA are optional; do not add either to satisfy an example. A client directive belongs only at a host-framework boundary that understands it; it is not required by React itself.

## Decision Table

| Need | Repository-native choice |
|---|---|
| Native button, input, or link | Semantic HTML with its actual DOM props |
| Popup, selection, composite keyboard/focus behavior | Reuse the installed accessible primitive |
| Polymorphic element | Preserve the installed `render`, `asChild`, or other public contract |
| Rearrangeable semantic regions | Named compound parts following local exports |
| Repeated visual dimensions | Existing variant tooling, or plain conditional classes |
| Parent layout depends on child presence | Existing slot/CSS conventions when they clarify ownership |
| One-off feature | Compose primitives; avoid expanding the generic API |

## Versioned Ref Forwarding

Choose **one** implementation matching the supported React runtime and React types. These are complete component modules, not styled application demos. Both preserve caller events, refs, native attributes, and submit opt-in; styling and an accessible name come from the consumer. There is no Base UI, CVA, router, or alias dependency.

### React 19 module: `action-react19`

```tsx
import type { ComponentPropsWithRef } from 'react';

export function Action({
  type = 'button',
  ...props
}: ComponentPropsWithRef<'button'>) {
  return <button {...props} type={type} />;
}
```

The spread forwards `ref` in React 19. React 18 function components do not receive `ref` this way.

### React 18-compatible module: `action-react18`

```tsx
import { forwardRef, type ComponentPropsWithoutRef } from 'react';

export const Action = forwardRef<
  HTMLButtonElement,
  ComponentPropsWithoutRef<'button'>
>(function Action({ type = 'button', ...props }, ref) {
  return <button {...props} type={type} ref={ref} />;
});
```

A library supporting both React 18 and 19 may retain `forwardRef`; React 19 support alone is not a reason to break older consumers. Do not render both modules under the same export.

## Primitive Wrappers and Composition

When wrapping Base UI or another installed primitive:

- Inherit its real props and ref type; do not replace them with a hand-written DOM approximation.
- Preserve all controlled/default state props and change callbacks.
- Forward refs to the actual interactive element. Merge internal and caller refs through the project's compatible helper; callback refs may have React 19 cleanup semantics.
- If internal behavior is added, compose rather than overwrite caller handlers. Respect the primitive's documented cancellation mechanism; do not assume every library uses `defaultPrevented` identically.
- Preserve function-valued `className` where the primitive supports it; feeding such a callback directly to a string-only `cn()`/CVA helper loses its state contract.
- Merge caller classes using existing tooling. Add variants only for meaningful dimensions; existing CVA users can colocate definitions and infer `VariantProps`.
- Preserve labels, IDs, ARIA relationships, disabled semantics, form integration, state attributes, portal/focus behavior, and polymorphic props. A link masquerading as a button does not inherit native disabled behavior.
- Use semantic tokens and fixed `data-slot` hooks only where established. Do not spread primitive-only props onto a raw DOM node.

Custom render components must pass the primitive-provided events, ref, class/state attributes, and children through to the intended DOM node. Confirm the installed primitive's merge utilities and exact signature rather than copying a newer `useRender` API.

## State Ownership and Effects

- A controlled `value`/`checked`/`open` comes from the caller; changes notify it. An uncontrolled component initializes from `defaultValue`/`defaultChecked`/`defaultOpen` and owns subsequent updates. Do not pass both, switch modes during a component's lifetime, or treat `false`, `0`, or an empty string as “uncontrolled.”
- Use the existing primitive's state management when available. Do not mirror controlled props in local state via effects or fire change callbacks from effects.
- Compute derived values while rendering. Handle user-triggered mutations in event handlers, not render/effect chains. Effects synchronize external systems; include correct dependencies and symmetric cleanup for subscriptions, listeners, timers, and requests.
- Prefer minimal local state; lift it only to the nearest actual owner. Use stable keys for list identity and intentional state reset, not indexes for reorderable records or random values.
- Avoid blanket `memo`, `useMemo`, and `useCallback`. Measure a real bottleneck and inspect whether the repository enables React Compiler before adding memoization.

## Loading and Host Boundaries

Use the existing router/framework loader or query integration for route data, pending/error states, cancellation, and invalidation. Start independent reads together at their owning boundary rather than making children wait for parent effects to begin fetching; keep truly dependent requests sequential. Do not add a cache/query dependency merely to implement this rule.

If a client effect must fetch, handle stale responses/unmount cleanup, loading/error states, and HTTP failure status explicitly. Avoid duplicate sources of truth with an existing query cache. Mutations and paid/non-idempotent calls must not be replayed by mount effects or automatic retries without an explicit idempotency/authorization contract.

`'use client'`, Next-specific images/navigation, React Server Components, server actions, and framework cache APIs are not universal React features. Do not transplant them into TanStack, Vite, or Cloudflare projects. Keep server-only secrets out of browser modules and use the actual host's server boundary; avoid Node/Bun assumptions in Worker/browser code.

## Compound Components

Expose parts when users need to rearrange or omit meaningful regions. A family such as `Card`, `CardHeader`, `CardTitle`, `CardDescription`, `CardHeaderAction`, `CardBody`, and `CardFooter` is preferable to a root with `title`, `description`, `icon`, `action`, and layout flags.

The parts should:

- inherit the rendered element or installed primitive's props;
- own one semantic role each;
- expose stable family-prefixed slots where the repository uses slot styling;
- share context only when behavior requires shared state;
- remain usable in more than one valid composition.

Do not create compound parts for incidental markup that callers should never arrange independently.

## Contextual Styling

Where Tailwind v4 and Base UI are installed, Selia-style selectors such as `*:data-[slot=...]`, `**:data-[slot=...]`, `has-[...]`, and primitive state attributes can express relationships. Check the local Tailwind and primitive versions; state names differ by component and release.

Use them when the parent legitimately owns layout between known children—for example, shifting a description when an icon slot is present. Keep interaction state in the headless primitive. Avoid JS booleans that merely restate DOM presence, but prefer explicit props when the value is part of the public behavioral contract.

## API and Styling Details

- Use semantic tokens supplied by the project, including foreground, muted, card, popover, border, danger, and focus-ring tokens.
- If CVA is already appropriate, group long definitions by concern; short definitions may remain strings.
- Forward `children` as part of the public contract; destructure it only when positioning or transforming it is required.
- Preserve local variant names; a chrome-free `plain` variant is useful only when composition actually needs it.
- Keep defaults opinionated and finished, while allowing `className` overrides.
- With Tailwind, follow the installed scale/tokens; with other styling tools, preserve their conventions instead.

## Common Mistakes

| Mistake | Correction |
|---|---|
| Reimplementing popup, selection, focus, or keyboard state | Reuse the installed accessible primitive; do not mandate a new library |
| Building a new context around state already owned by a primitive | Compose or expose the primitive's existing parts and callbacks |
| Adding Base UI to a static presentational wrapper | Use semantic HTML |
| Adding CVA to a component with no variants | Reuse the existing class helper or plain classes |
| Adding `data-slot` to every nested DOM node | Mark public roots and meaningful compound parts |
| Replacing the existing polymorphism API with a preferred one | Preserve the installed contract, including events and refs |
| Growing a root component with icon/action/layout flags | Split meaningful, rearrangeable regions into named parts |
| Copying Selia tokens or aliases into a different project | Map the pattern to the target's existing tokens and paths |
| Adding registry, docs, dependencies, or barrel exports without request | Keep the change scoped to the component and its existing integration path |
| Treating an upstream quirk as a rule | Confirm repetition and purpose across nearby source files |

## Verification

Use the repository's existing formatter/typecheck/focused tests and a proportionate build when authorized. Exercise ref focusing, caller handlers/cancellation, form submit behavior, controlled/uncontrolled updates, keyboard navigation, focus restoration, disabled/invalid states, ARIA relationships, portal layering, and relevant themes. Confirm loading/error/stale-response behavior when data flow changes. Report commands actually run; static code/class review is not evidence of visual or keyboard parity.

## Sources and Limits

- [React `forwardRef`](https://react.dev/reference/react/forwardRef): React 18-compatible forwarding and React 19 ref props.
- [React input contracts](https://react.dev/reference/react-dom/components/input): controlled/uncontrolled values and events.
- [React effects guidance](https://react.dev/learn/you-might-not-need-an-effect): derived state and event logic.
- [Base UI composition](https://base-ui.com/react/handbook/composition): consult the installed release before using merge/render utilities.
- [Vercel agent skills](https://github.com/vercel-labs/agent-skills): consulted for general waterfall, bundle, and UI principles only. Its Next.js/Vercel-specific rules are not requirements here.

These links provide rationale, not dependency/version mandates or adoption/license claims. The example modules need matching React/React DOM/types, TypeScript JSX configuration, and a DOM-capable host; neither proves accessibility of its consuming application.
