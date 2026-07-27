---
name: astro
description: Enforce Astro 7 static-first conventions. Use when editing .astro files or when the user mentions Astro, islands, hydration, client directives, server:defer, content collections, or getCollection. Ships zero JS by default and picks the lightest client directive that works — client:visible or client:idle ahead of client:load, getCollection ahead of Astro.glob.
paths:
  - "**/*.astro"
allowed-tools:
  - Read
  - Grep
---

> Targets Astro 7 · verified 2026-07 (latest 7.1.3).

This skill enforces Astro static-first conventions. The rule: render to static HTML by default, ship JavaScript only where there is real interactivity, and pick the lightest hydration directive that works.

Apply only when the project uses `astro ^7.0` or higher. If `package.json` pins an older major, **STOP** and ask before applying.

## What changed in 7 (and 6)

Client directives, content collections and server islands are unchanged — the rules below carry over. What did change:

| Version | Change | What it means |
|---|---|---|
| 7 | Rust compiler replaces the Go one | Every non-void element needs a closing tag. Invalid markup is no longer auto-corrected — it errors |
| 7 | `compressHTML` defaults to `'jsx'`, not `true` | Whitespace between inline elements is stripped by JSX rules. Check spacing-sensitive layouts after upgrading |
| 7 | Markdown runs on Sätteri instead of remark/rehype | Drop `@astrojs/markdown-remark`, or install it explicitly if you depend on specific plugins |
| 7 | `src/fetch.ts` is a reserved filename | Rename it, or point `fetchFile` elsewhere |
| 7 | Vite 8 | Re-check Vite plugins and config |
| 6 | Node 18 and 20 dropped | Requires Node 22.12+ |
| 6 | The v2-era Content Collections API is gone | Content Layer `loader` API only; config must live at `src/content.config.ts` |
| 6 | `import.meta.env` values are always inlined | Automatic type coercion is gone — parse strings yourself |

## Core principles

- **Default to zero JS**. Astro components render to HTML at build (or request) time and ship no JS. Adding `client:*` to a component opts into hydration—do it deliberately.
- **Pick the lightest `client:*` that works**. The directive is a hydration trigger, not a rendering choice.
- **Server islands (`server:defer`)** for personalized content within an otherwise static page. Use this instead of switching the whole page to SSR.
- **Content Collections** for typed markdown/MDX content. Do not hand-roll glob imports for blogs/docs.

## client:* hierarchy

Pick the first directive that fits, top to bottom:

| Directive | When to use | JS cost |
|---|---|---|
| (none) | Pure presentational HTML/CSS | 0 |
| `client:visible` | Interactive but below the fold (footer widgets, late-page forms) | Loaded on viewport intersection |
| `client:idle` | Interactive but not critical (search, login) | Loaded after page idle |
| `client:media="(...)"` | Only on certain viewports (mobile menu) | Loaded conditionally |
| `client:load` | Above-the-fold interactive (live counter, top-nav search) | Loaded eagerly, blocks |
| `client:only="<framework>"` | Component cannot SSR (relies on `window` at render time) | Skips SSR entirely |

`client:load` is the heaviest. Default to `client:visible` or `client:idle`, escalate only when there is a measurable delay the user notices.

## Forbidden patterns

- `client:load` on a component with no `useState` / no event handlers / no live data. If it is presentational, drop the directive.
- `client:load` on every interactive component when `client:visible` would do.
- Wrapping the entire page in a single `<Layout client:load>`. Hydrate per-island, not per-page.
- Importing a React/Vue/Svelte component into `.astro` without realizing the hydration cost—every framework adds runtime weight.
- Server-side data fetching inside an island that re-fetches on every hydration. Move the fetch to the parent `.astro` and pass as prop.
- `Astro.props` mutated inside the component. Treat as immutable.
- `getStaticPaths` returning thousands of paths without pagination or filtering. Build time grows.
- `Astro.glob('../posts/*.md')`. Deprecated in Astro v5+. Use Content Collections (`getCollection()`) for content/markdown with typed schemas, or `import.meta.glob()` (with `eager: true` if you need synchronous import) for other file types.
- Mixing `output: 'static'` with `client:load` on dynamic components that need fresh data per request. Use `server:defer` or set `prerender = false` for that route.
- A new island / component when the project already has an equivalent under `src/components/`. grep first; reuse or extend if found.

## Server islands

```astro
---
import UserGreeting from '../components/UserGreeting.astro'
---
<html>
  <body>
    <h1>Welcome</h1>
    <UserGreeting server:defer>
      <p slot="fallback">Loading…</p>
    </UserGreeting>
    <main>... static content ...</main>
  </body>
</html>
```

The page ships static HTML immediately; the personalized greeting renders on the server in parallel and streams in. No SSR for the whole page.

## Content Collections

Astro 5+ uses the `loader` API in `src/content.config.ts` (note the new file path—not `src/content/config.ts`).

```ts
// src/content.config.ts
import { defineCollection, z } from 'astro:content'
import { glob } from 'astro/loaders'

const blog = defineCollection({
  loader: glob({ pattern: '**/*.md', base: './src/content/blog' }),
  schema: z.object({
    title: z.string(),
    pubDate: z.date(),
    tags: z.array(z.string()).default([])
  })
})

export const collections = { blog }
```

```astro
---
import { getCollection } from 'astro:content'
const posts = await getCollection('blog')
---
```

Do not use:
- `Astro.glob('../posts/*.md')` — deprecated in v5+. For non-content files, use `import.meta.glob()` instead.
- `type: 'content'` / `type: 'data'` — replaced by the `loader` API.
- `src/content/config.ts` — the file moved to `src/content.config.ts`.

## When you need full SSR

If the page genuinely needs per-request data for the entire layout (signed-in dashboard root), set `export const prerender = false` on that route. Reach for `server:defer` first; full SSR only when the entire shell depends on request context.

## When the right hydration is unclear

> Component [X] is interactive [how]. Above the fold? [yes/no]. Critical for first paint? [yes/no]. Recommended directive: [...]. Approve?

## Verification (grep after every .astro change)

```bash
grep -rnE 'client:load' --include='*.astro' .                   # can it be downgraded to visible/idle?
grep -rnE "Astro\.glob\(" --include='*.astro' --include='*.ts' .   # replace with getCollection
grep -rnE '<Layout[^>]*client:' --include='*.astro' .           # don't put client:* on Layout
```

Reference: https://docs.astro.build/en/concepts/islands/
