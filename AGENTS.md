# AGENTS.md — skills

Personal agent skills (`v1-*`): writing guides for Astro, Cloudflare, Elysia, Express, FastAPI, Go, Laravel, Python, React, TypeScript.

## How to work

1. **Think before coding.** State your assumptions. If there are several interpretations, name them instead of picking one silently. Ask only when the answer cannot be found in the code or the repo docs; if it can be inferred, proceed.
2. **Simplicity first.** Write the minimum code that solves the request. No features, abstractions, or configuration that were not asked for. If 200 lines could be 50, rewrite it.
3. **Surgical changes.** Touch only what the task needs. Don't "improve" adjacent code, comments, or formatting. Match the existing style. If you notice unrelated dead code, mention it; don't delete it. Remove only the imports/variables your own change made unused.
4. **Goal-driven execution.** Define a check for "done": for a bug, write a test that reproduces it, then make it pass. For multi-step work, state a short plan as `step -> verify: check`. If you cannot run the tests, say so; never claim they passed.

## Code structure

- One folder per skill, named `v1-writing-<stack>`, with a `SKILL.md` (frontmatter `name` and `description`, then the guide). Follow the format of the existing skills.
- Skills follow YAGNI/KISS and the Karpathy principles above; don't add rules that push agents toward extra layers or speculative abstractions.
- When you add or rename a skill, update the catalog in `README.md`.

## This project

- Plain Markdown; no build step.
