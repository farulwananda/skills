---
name: fw-writing-astro
description: Use when building, reviewing, or refactoring Astro components, content-driven sites, static pages, or on-demand routes. Covers rendering boundaries, minimal browser scripts, typed props, version-aware Content Collections, locale routing, metadata, and assets.
license: MIT
metadata:
  author: Farul Wananda Astro conventions; modern web standards
---

# FW Writing Astro

## Scope and first decisions

Use this skill for Astro-specific components, routing, content, rendering, and integrations. Do not use it to prescribe React internals, a backend architecture, or a deployment provider; pair the relevant framework/runtime skill only when needed.

Before editing, inspect package.json, the lockfile, Astro and adapter versions, astro.config.*, tsconfig, existing routes/layouts/content configuration, and repository scripts. Follow installed-version documentation and existing conventions. Do not install integrations, upgrade dependencies, deploy, or call paid services without authorization.

- Prefer prerendered HTML for content that can be built ahead of time. Frontmatter runs at build time for prerendered routes, and on the server for on-demand routes; it is not browser code.
- Use SSR for actual request-time requirements such as authenticated content or fresh data. It requires a compatible adapter. In versions supporting the current model, default static output can opt individual routes into SSR with `export const prerender = false`; `output: 'server'` makes SSR the default and permits `prerender = true` exceptions. Older versions have different hybrid configuration: inspect before migrating.
- A fully static site normally needs no adapter. Do not equate Astro with Cloudflare or impose SSR on static content.
- Preserve accessibility, escaping, and trust boundaries. Do not interpolate untrusted HTML with `set:html`; sanitize trusted-format rendering at its actual input boundary.

## Native components and valid props

Use `.astro` components for layout and static content. Declare `Props` when a component accepts props; prop-free pages need no ceremonial interface. Use the actual element's attributes, forward attributes deliberately, and avoid a polymorphic tag with incompatible attribute sets. A linked card can use a link inside an article instead of sending `href` to an article. Do not nest interactive controls inside a wrapping anchor.

**ASTRO-CARD** — complete `src/components/Card.astro`; no CSS library required:

```astro
---
import type { HTMLAttributes } from 'astro/types';

interface Props extends Omit<HTMLAttributes<'article'>, 'title'> {
  title: string;
  description?: string;
  href?: string;
}

const { title, description, href, class: className, ...rest } = Astro.props;
---

<article {...rest} class:list={['card', className]}>
  <h2>{href ? <a href={href}>{title}</a> : title}</h2>
  {description && <p>{description}</p>}
  <slot />
</article>
```

Use named slots for explicit content regions and `class:list` for conditional classes. Validate untrusted link destinations separately; a string type is not URL validation.

## Small browser behavior before framework islands

Use semantic HTML first (`details`, native forms), then a small processed `<script>` when browser behavior is needed. Do not add React/Vue/Svelte just to toggle a menu. Astro scripts are bundled/deduplicated; select all component instances or use a scoped custom element. With client-side navigation, follow the installed router's lifecycle and avoid duplicate listeners. Frontmatter values are not automatically available in browser scripts; pass only non-secret data through explicit DOM attributes or props.

**ASTRO-BUTTON** — complete `src/components/Counter.astro`, for ordinary full-page navigation:

```astro
<button type="button" data-counter aria-label="Increment count">0</button>
<script>
  document.querySelectorAll<HTMLButtonElement>('button[data-counter]').forEach((button) => {
    let count = 0;
    button.addEventListener('click', () => {
      button.textContent = String(++count);
    });
  });
</script>
```

Use a framework island when existing framework components or substantial interactive state justify it, with an already installed integration. Choose `client:load` for immediately needed behavior, `client:idle` for noncritical behavior, `client:visible` for offscreen content, and `client:media` for an actual media-dependent requirement. Components without a `client:*` directive render HTML without framework hydration. `client:only` skips server rendering and needs an intentional loading/fallback experience, not a blanket workaround.

## Content Collections follow the installed version

Use collections for repeated structured content, not every isolated page. Inspect existing configuration and entry identifiers before editing:

- Astro 5 introduced the Content Layer loader model, normally in `src/content.config.ts`. Legacy projects may use `src/content/config.ts` and `type: 'content'` rather than loaders. These are not interchangeable file locations/APIs.
- Check the installed Zod export. The following example targets Astro 5's `z` export from `astro:content`; current documentation uses `astro/zod`. Do not mechanically replace imports in a version that lacks that module.
- Loader entries use `id`; do not assume legacy `slug` or `entry.render()` APIs. For Content Layer rendering, use the supported `render(entry)` helper. MDX requires its integration if MDX files are included.

**ASTRO-COLLECTION-V5** — complete `src/content.config.ts` for Astro 5 with local Markdown entries:

```typescript
import { defineCollection, z } from 'astro:content';
import { glob } from 'astro/loaders';

const blog = defineCollection({
  loader: glob({ pattern: '**/*.md', base: './src/content/blog' }),
  schema: z.object({
    title: z.string(),
    description: z.string(),
    pubDate: z.coerce.date(),
    updatedDate: z.coerce.date().optional(),
    tags: z.array(z.string()).default([]),
    draft: z.boolean().default(false),
  }),
});

export const collections = { blog };
```

**ASTRO-COLLECTION-LIST** — complete page `src/pages/blog/index.astro`, paired with that collection and at least one schema-valid Markdown entry:

```astro
---
import { getCollection } from 'astro:content';
const posts = await getCollection('blog', ({ data }) => !import.meta.env.PROD || !data.draft);
posts.sort((a, b) => b.data.pubDate.valueOf() - a.data.pubDate.valueOf());
---
<h1>Blog</h1>
<ul>{posts.map((post) => <li>{post.data.title}</li>)}</ul>
```

This list deliberately does not invent detail-page URLs. If adding links, implement matching dynamic routes and `getStaticPaths()` for prerendered pages. Check drafts at detail routes as well as listings; filtering only the index is not access control. Request-time private content must not leak through build output or public caches.

## Locale, canonical, and 404 consistency

Choose the URL policy before changing routes: configured production `site`, deployment `base`, trailing slash/build format, locale prefixes, root behavior, and fallback behavior must agree with actual files and host redirects.

- Use folder-based routes. With `prefixDefaultLocale: false`, default-language pages live at the root; other locales have folders. With `true`, all languages have locale folders and the root index still needs intentional handling.
- Use version-supported `astro:i18n` helpers when useful. Configuration does not translate content or create missing pages. Link only existing translations; preserve a meaningful equivalent path when switching languages.
- Canonicals should use the production origin and the chosen canonical pathname, not arbitrary request host/query parameters from `Astro.url.href`. Set `site` for the deployment; account for `base` and trailing-slash normalization. Set document `lang` and emit `hreflang` alternatives only for real equivalents with reciprocal links. Do not canonicalize all locales to English.
- Implement `src/pages/404.astro` when a custom error page is needed. On SSR paths return a real 404 status, not a successful error body. On static hosting verify the generated 404 file is served with status 404. Do not use SPA fallback for a multi-page site or redirect every missing locale path to the homepage. Avoid indexing error pages and do not give every missing URL a self-canonical.

**ASTRO-CANONICAL-FRAGMENT** — partial layout example, not a standalone page. Assumes `site` is configured and the current pathname already obeys the repository's base/locale/slash policy:

```astro
---
if (!Astro.site) throw new Error('Configure the production site URL');
const canonical = new URL(Astro.url.pathname, Astro.site);
---
<link rel="canonical" href={canonical} />
```

## Assets and deployment boundaries

Prefer `src/assets/` and `Image` from `astro:assets` for images that benefit from processing. Keep assets requiring unchanged URLs, robots.txt, and icons in `public/`; ordinary `<img>` is valid for deliberate passthrough/remote cases. Supply meaningful `alt` (empty for decorative images) and intrinsic dimensions or equivalent layout constraints. Match responsive-image APIs to the installed version and avoid eagerly loading every image; prioritize only genuinely above-the-fold critical images. Check SSR image services and their runtime/cost requirements before enabling them.

Use the existing host's headers/redirect mechanisms only when needed. `_headers` and `_redirects` are provider-specific, not universal Astro requirements. For adapter SSR, follow that adapter version's generated output and runtime binding conventions rather than a hand-written generic static/SPA configuration.

## Verification and reporting

Use the repository's existing check/build/test scripts when authorized. Check prop typing, emitted routes, collection validation, draft detail-page exposure, locale switch links, canonical origin/query removal, base/slash behavior, unknown-route status, keyboard behavior, and browser console errors. A successful static build does not prove SSR runtime compatibility or production redirects. Report commands actually run, version assumptions, missing integrations/content fixtures, and unverified hosting behavior. Do not install, deploy, or trigger remote services as an implicit verification step.

## Sources and provenance

This is the author's synthesis, not a bulk copy of third-party skills. Read only the documentation relevant to the task and installed version:

- Official Astro: [components and props](https://docs.astro.build/en/basics/astro-components/), [browser scripts](https://docs.astro.build/en/guides/client-side-scripts/), [on-demand rendering](https://docs.astro.build/en/guides/on-demand-rendering/).
- Official Astro: [Content Collections](https://docs.astro.build/en/guides/content-collections/), [Astro 5 migration](https://docs.astro.build/en/guides/upgrade-to/v5/), [i18n routing](https://docs.astro.build/en/guides/internationalization/), [Cloudflare adapter](https://docs.astro.build/en/guides/integrations-guide/cloudflare/).
- [astrolicious/agent-skills](https://github.com/astrolicious/agent-skills) is a community reference, not official Astro documentation or a compatibility guarantee. Consult it selectively; official version-matched API documentation takes precedence.
