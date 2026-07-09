---
name: sveltekit-config
description: SvelteKit 配置/构建/部署/性能技能。当用户配置 adapter（node/static/cloudflare/netlify/vercel）、使用 advanced routing/layouts、优化性能（代码分割/asset/hydration）、处理 images（@sveltejs/enhanced-img）、实现 accessibility/SEO、调试 SvelteKit 应用、从 SvelteKit v1/Sapper 迁移时使用。
---

# SvelteKit Config

## When to use this skill

Use this skill whenever you are working on SvelteKit's configuration, build pipeline, deployment, performance, images, accessibility, SEO, debugging, or migration concerns. This includes:

- Choosing or configuring an adapter (`adapter-node`, `adapter-static`, `adapter-cloudflare`, `adapter-netlify`, `adapter-vercel`)
- Building (`vite build`) and previewing (`vite preview`) production apps
- Advanced routing (rest params, optional params, matchers, route sorting, filename encoding)
- Advanced layouts (route groups, breaking out of layouts with `@`)
- Auth integration patterns (sessions vs tokens, hooks-based auth)
- Performance optimization (code splitting, asset opt, avoiding waterfalls)
- Image handling (`@sveltejs/enhanced-img`, CDN loading, `<picture>`/`<img>` patterns)
- Accessibility (route announcements, focus management, `lang` attribute)
- SEO (titles, meta tags, sitemaps, JSON-LD)
- Breakpoint debugging (VS Code, Chrome DevTools)
- Migrating from SvelteKit v1 or Sapper

For routing/data/load basics, see `sveltekit-overview`. For Svelte components/runes, see `svelte-runes`.

## Critical: Building and previewing

`vite build` runs in two stages: Vite produces an optimized production build, then your adapter tailors the output for the target platform. Prerendering executes during build.

During the build, SvelteKit loads your `+page/layout(.server).js` for analysis. Code that must NOT run at build time should guard with `building` from `$app/environment`:

```js
import { building } from '$app/environment';
import { initialiseDatabase } from '$lib/server/database';

if (!building) initialiseDatabase();

export function load() { /* ... */ }
```

After building, run `vite preview` (or `npm run preview`) to test the production build locally. Preview runs in Node, so adapter-specific behavior (e.g. Cloudflare's `platform` object) does NOT apply — use `wrangler dev` for Cloudflare or the platform's CLI for accurate testing.

```sh
npm run build       # vite build + adapter
npm run preview     # vite preview (Node)
```

## Critical: Adapters — when to use which

The adapter is configured in `svelte.config.js` under `kit.adapter`. `adapter-auto` ships by default in new projects and picks the right adapter for known deployment environments (Cloudflare Pages, Netlify, Vercel, Azure SWA, SST, Google Cloud Run). Once you've chosen a target, install that adapter explicitly so it lands in your lockfile.

| Target                          | Adapter                          | Notes |
|---------------------------------|----------------------------------|-------|
| Node server / Docker / VM       | `@sveltejs/adapter-node`         | Standalone Node server. Most flexible. |
| Static hosting (no SSR)         | `@sveltejs/adapter-static`       | SSG or SPA fallback. |
| Cloudflare Workers / Pages      | `@sveltejs/adapter-cloudflare`   | Unified adapter for both. |
| Netlify                         | `@sveltejs/adapter-netlify`      | Node functions or Deno edge. |
| Vercel                          | `@sveltejs/adapter-vercel`       | Serverless or edge, ISR support. |

Adapter quick guide:

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-node';
export default { kit: { adapter: adapter() } };
```

### Platform-specific context

Some adapters expose platform info (KV namespaces, Durable Objects, env vars) via `event.platform` in hooks/server routes. Type augmentation in `src/app.d.ts`:

```ts
declare global {
  namespace App {
    interface Platform {
      env: { MY_KV: KVNamespace };
    }
  }
}
export {};
```

Always prefer `$env/static/private` for environment variables — `$env/dynamic/*` cannot be used during prerendering.

## Critical: Advanced routing

SvelteKit routes are filesystem-based. Beyond basic dynamic segments, you have several advanced features.

### Rest parameters — `[...rest]`

Match an unknown number of segments. `src/routes/a/[...rest]/z/+page.svelte` matches `/a/z`, `/a/b/z`, `/a/b/c/z`. The `rest` param is a string with `/`-separated segments.

```tree
src/routes/[org]/[repo]/tree/[branch]/[...file]/+page.svelte
```

Use rest parameters to render custom 404s — add `[...path]/+page.js` that calls `error(404)` so a nested `+error.svelte` is reached.

### Optional parameters — `[[lang]]`

Wrap with double brackets to make a param optional. `[[lang]]/home` matches both `/home` and `/en/home`. An optional param cannot follow a rest param.

### Matching — `[name=type]`

Constrain a parameter with a matcher from `src/params/`:

```js
// src/params/fruit.js
/** @type {import('@sveltejs/kit').ParamMatcher} */
export function match(param) {
  return param === 'apple' || param === 'orange';
}
```

Then write `src/routes/fruits/[page=fruit]/+page.svelte`. Matchers run on both server and browser.

### Sorting

When multiple routes match, SvelteKit sorts by:

1. More specific routes win (fewer params = more specific)
2. Matchers (`[name=type]`) beat unconstrained (`[name]`)
3. `[[optional]]` and `[...rest]` are lowest priority unless they're the final segment
4. Ties resolved alphabetically

### Encoding special characters

Use hex escape `[x+nn]` in folder names: `/` → `[x+2f]`, `:` → `[x+3a]`, etc. Use `[u+nnnn]` for Unicode (no surrogate pairs needed).

```tree
src/routes/smileys/[x+3a]-[x+29]/+page.svelte
# matches /smileys/:-)
```

## Critical: Advanced layouts

By default, the layout hierarchy mirrors the folder hierarchy. Use these patterns to reshape it.

### Route groups — `(group)`

Parentheses-wrapped folder names don't appear in the URL. Use to share a layout between routes without affecting URL structure.

```tree
src/routes/
├ (app)/dashboard/+page.svelte
├ (app)/+layout.svelte          # app shell
├ (marketing)/about/+page.svelte
├ (marketing)/+layout.svelte    # marketing shell
└ +layout.svelte
```

### Breaking out — `+page@layout.svelte`

Append `@<segment>` (or `@` for root) to reset the layout chain. `+page@(app).svelte` inherits only from `(app)/+layout.svelte`.

Options: `+page@[id].svelte`, `+page@item.svelte`, `+page@(app).svelte`, `+page@.svelte`.

Layouts can also break out: `+layout@.svelte` rewinds to root for everything below it.

### Reset to root

If you want most of your app under one layout but a few routes to escape, put everything inside a group except the outliers:

```tree
src/routes/
├ (app)/...
└ admin/+page.svelte   # does NOT inherit (app) layout
```

## Critical: Auth integration

Auth = authentication (who is this?) + authorization (what can they do?).

### Sessions vs tokens

- **Sessions**: ID stored in DB. Revocable instantly, requires DB lookup per request.
- **JWT tokens**: Self-contained, no DB lookup, but cannot be revoked immediately. Better latency.

### Integration pattern

Check auth cookies in `src/hooks.server.js`, populate `event.locals.user`, then read `locals` in `+page.server.js` / `+server.js` load functions.

```js
// src/hooks.server.js
export async function handle({ event, resolve }) {
  event.locals.user = await getUser(event.cookies.get('session'));
  return resolve(event);
}
```

### Libraries

- `npx sv add better-auth` — Better Auth integration via Svelte CLI
- [Lucia auth guide](https://lucia-auth.com/) — reference SvelteKit examples for session-based auth

Always require `path: '/'` when calling `cookies.set(...)` in SvelteKit v2.

## Critical: Performance optimization

SvelteKit ships with: code-splitting, asset preloading, file hashing, request coalescing, parallel loading, data inlining, conservative invalidation, link preloading. To go further:

### Diagnose

- PageSpeed Insights / Lighthouse / WebPageTest
- Chrome DevTools Network + Performance tabs
- Test in **preview mode** (after `vite build`), not dev mode

### Assets

- Use `@sveltejs/enhanced-img` for images (smaller formats, intrinsic dimensions)
- Lazy-load below-the-fold videos with `preload="none"`
- Subset fonts; preload critical fonts via `handle` hook's `preload` filter

### Code size

- Use Svelte 5 (smaller than 4)
- Use `rollup-plugin-visualizer` to find heavy packages
- Prefer dynamic `import()` for conditional code
- Push third-party scripts to web workers (Partytown)

### Avoid waterfalls

- Use **server `load` functions** for backend calls (avoid client → server → backend chains)
- Issue parallel queries with `Promise.all` / DB joins
- SPA mode causes waterfalls — prerender instead

### Hosting

- Deploy frontend near backend (or use edge)
- Ensure HTTP/2+

## Critical: Images

### Vite's built-in handling

```svelte
<script>
  import logo from '$lib/assets/logo.png';
</script>
<img alt="logo" src={logo} />
```

Vite hashes the filename and inlines small assets.

### @sveltejs/enhanced-img

Build-time image optimization: generates `avif`/`webp`, sets intrinsic `width`/`height` (prevents CLS), strips EXIF.

```js
// vite.config.js — plugin order matters
import { enhancedImages } from '@sveltejs/enhanced-img';
import { sveltekit } from '@sveltejs/kit/vite';
export default { plugins: [enhancedImages(), sveltekit()] };
```

Usage:

```svelte
<enhanced:img src="./image.jpg" alt="..." sizes="min(1280px, 100vw)" />
```

Generated `<picture>` includes multiple formats and sizes for HiDPI. Provide 2x source for retina displays.

Custom widths: `<enhanced:img src="./image.png?w=1280;640;400" />`
Per-image transforms: `<enhanced:img src="./image.jpg?blur=15" />`

### Dynamic CDN loading

For images unavailable at build time (CMS, DB), use a CDN library:

- `@unpic/svelte` — CDN-agnostic
- `svelte-cloudinary` — Cloudinary
- CMS-bundled: Contentful, Storyblok, Contentstack

### Best practices

- Set `fetchpriority="high"` and avoid `loading="lazy"` for LCP images
- Always provide `alt` text
- Don't use `em`/`rem` in `sizes`
- Mix strategies: Vite for `<meta>`, enhanced-img for hero, CDN for user content

## Critical: Accessibility

SvelteKit provides an accessible foundation; you're still responsible for app-level a11y.

### Route announcements

SvelteKit injects a live region that reads the `<title>` after each client-side navigation. Every page must have a unique, descriptive `<title>` in a `<svelte:head>`:

```svelte
<svelte:head>
  <title>Todo List</title>
</svelte:head>
```

### Focus management

After each navigation, SvelteKit focuses `<body>` (or `[autofocus]` element if present). Override with `afterNavigate` for custom behavior:

```js
import { afterNavigate } from '$app/navigation';
afterNavigate(() => document.querySelector('.focus-me')?.focus());
```

Use `data-sveltekit-keepfocus` on a `<form>` to preserve input focus. `goto(url, { keepFocus: true })` for programmatic nav.

### `lang` attribute

Set `<html lang="en">` (or your language) in `src/app.html`. For multi-language sites, use a `transformPageChunk` in `handle` to set per-request.

## Critical: SEO

SvelteKit ships with SSR, normalized trailing-slash URLs, and good defaults. Manual steps:

### Per-page meta

```svelte
<svelte:head>
  <title>Page Title — Site Name</title>
  <meta name="description" content="..." />
  <meta property="og:title" content="..." />
  <meta property="og:description" content="..." />
  <meta property="og:image" content="..." />
  <meta property="og:type" content="website" />
  <link rel="canonical" href="https://..." />
</svelte:head>
```

Common pattern: return SEO data from `load`, render in root layout's `<svelte:head>`.

### Sitemap

```js
// src/routes/sitemap.xml/+server.js
export async function GET() {
  return new Response(`<?xml version="1.0" encoding="UTF-8"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
      <!-- <url> entries -->
    </urlset>`, { headers: { 'Content-Type': 'application/xml' } });
}
```

### JSON-LD structured data

Inject via `<svelte:head>` with `<script type="application/ld+json">`.

## Critical: Migration

### SvelteKit v1 → v2

Use `npx sv migrate sveltekit-2` (or `npx svelte-migrate` for older projects). Breaking changes:

- `error(...)` and `redirect(...)` are no longer thrown — just call them
- `cookies.set/delete/serialize` requires `path: '/'`
- Top-level promises in `load` are NOT awaited automatically — use `await` explicitly
- `goto` no longer accepts external URLs (use `window.location.href`)
- `paths` are now consistently relative (default `true`)
- `$app/stores` deprecated in 2.12 — migrate to `$app/state` (Svelte 5 runes)
- Svelte 4 required, Node 18.13+
- `vitePreprocess` must be imported from `@sveltejs/vite-plugin-svelte`

### Sapper → SvelteKit

- `package.json`: add `"type": "module"`, remove `polka`/`sapper`/`sirv`/`compression`
- Add `@sveltejs/kit` + an adapter
- Scripts: `sapper build` → `vite build`, `sapper dev` → `vite dev`, `node __sapper__/build` → `node build`
- `src/template.html` → `src/app.html` (replace `%sapper.*` placeholders)
- Routes: `routes/about/index.svelte` → `routes/about/+page.svelte`
- `_layout.svelte` → `+layout.svelte`, `_error.svelte` → `+error.svelte`
- `preload` → `load` (different API: single `event` arg, no `this.fetch`)
- `stores` / `navigating` from `$app/stores` (or `$app/state` in 2.12+)
- Regex routes → matchers in `src/params/`
- `sapper:prefetch` → `data-sveltekit-preload-data`
- `sapper:noscroll` → `data-sveltekit-noscroll`
- Move internal libs from `src/node_modules/...` to `src/lib`

## Critical: Debugging

### VS Code

Built-in debug terminal works out of the box:

1. `CMD/Ctrl+Shift+P` → "Debug: JavaScript Debug Terminal"
2. Run `npm run dev` in that terminal
3. Set breakpoints, hit them in the browser

Or create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    { "command": "npm run dev", "name": "dev", "request": "launch", "type": "node-terminal" }
  ]
}
```

### Chrome DevTools / Edge

```sh
NODE_OPTIONS="--inspect" npm run dev
```

Open the site, then click the "Open dedicated DevTools for Node.js" icon (Node logo, top-left). Or visit `chrome://inspect`.

## Quick Fixes

- **`building` flag**: Wrap one-time init code in `if (!building)` to prevent it running during `vite build`.
- **`fallback` in `adapter-static`**: Use `200.html` for SPA mode, but avoid `index.html` (conflicts with prerendered `/`).
- **GitHub Pages**: Set `paths.base` to repo name and `fallback: '404.html'`. Add `.nojekyll` to `static/`.
- **Node origin**: Set `ORIGIN` env var if you're behind a proxy and see "Cross-site POST form submissions are forbidden".
- **XFF depth**: Behind N proxies, set `XFF_DEPTH=N` so `getClientAddress()` returns the real client IP.
- **Compression**: Use `@polka/compression` not `compression` — SvelteKit streams responses.
- **`.env` in production**: Production doesn't auto-load `.env`. Use `node --env-file=.env build` (Node 20.6+) or `-r dotenv/config`.
- **`platform` only in dev/preview**: Test Cloudflare/Netlify/Vercel platform APIs with their respective CLIs (`wrangler dev`, `netlify dev`, `vercel dev`).
- **`fs` not available**: Use `read` from `$app/server` (works in edge by fetching from deployed assets).
- **`goto` external URL**: Use `window.location.href` instead.
- **Cookie path**: SvelteKit v2 requires `path: '/'` on `cookies.set(...)`.
- **Trailing slash**: Set `trailingSlash: 'always'` if your host doesn't serve `/a.html` from `/a`.

## Gotchas

- **Preview is not production**: `vite preview` runs in Node, doesn't emulate adapter-specific behavior. Use the platform's CLI for accurate local testing.
- **`adapter-auto` is just for zero-config**: Once you've decided on a target, install the real adapter so it lands in the lockfile and you can pass options.
- **Server bundle size on Cloudflare**: Workers have a size limit. Move large libraries to client-only imports if you hit it.
- **AWS / Node 18**: SvelteKit v2 requires Node 18.13+. Older Node versions fail with cryptic errors.
- **ISR + prerender**: ISR has no effect on routes with `export const prerender = true`.
- **`adapter-cloudflare-workers` deprecated**: Use `@sveltejs/adapter-cloudflare` (with `assets.directory` + `assets.binding` in wrangler config).
- **Svelte 4 → 5**: `$app/stores` is deprecated; use `$app/state`. Update Svelte first, then SvelteKit.
- **Tailwind + `<enhanced:img>`**: Tag-name selectors need `enhanced\:img` to escape the colon.
- **Match `data-sveltekit-noscroll` for chat/SPA-like UIs**: Default scroll-to-top behavior can break infinite-scroll apps.
- **Vite asset inlining**: Assets below `assetsInlineLimit` (default ~4kb) get base64-inlined. Excludes `svg` for enhanced-img.

## FAQ

**Should I use `adapter-auto`?**
Yes for prototyping. Once you've chosen a target, swap to the specific adapter so you can configure it.

**SPA mode vs full SSR?**
Full SSR with `adapter-static` (prerender) is best for SEO/perf. SPA fallback is for when you must deploy to static-only hosting without prerendering everything. SPA hurts SEO and performance.

**When should I prerender?**
Pages that return the same content for every visitor (marketing pages, blog posts, docs). Add `export const prerender = true` to the route.

**How do I know which adapter is being used in production?**
`adapter-auto` logs at build time which one it picked. Otherwise, check `svelte.config.js`.

**Can I use the same `svelte.config.js` for multiple adapters?**
No — you must run `vite build` once per adapter. Common CI pattern: build matrix per target.

**Why is my Cloudflare Worker huge?**
Bundle bloat — server-side imports pull in large deps. Move them to dynamic `import()` or client-only.

**Do I need Svelte 5 for SvelteKit 2?**
You need Svelte 4+. Svelte 5 is recommended for `$app/state` and runes.

## Examples & References

### Examples

| File | What it covers |
|------|----------------|
| `examples/build-preview.md` | `vite build`, `vite preview`, env vars at build time |
| `examples/adapter-node.md` | dev/build/deploy, custom server, env vars |
| `examples/adapter-static.md` | prerender all, SPA fallback, GitHub Pages |
| `examples/adapter-cloudflare.md` | workers, pages, runtime APIs, env vars |
| `examples/adapter-netlify.md` | deploy, edge functions, env vars |
| `examples/adapter-vercel.md` | deploy, image opt, ISR, env vars, skew protection |
| `examples/writing-adapter.md` | custom adapter using the builder API |
| `examples/advanced-routing.md` | rest params, optional, matchers, sort, encoding |
| `examples/advanced-layouts.md` | nested layouts, named layouts, error reset, groups |
| `examples/performance.md` | code splitting, asset opt, hydration opt |
| `examples/images.md` | enhanced-img, dynamic CDN loading, best practices |
| `examples/accessibility.md` | route announcements, focus management, lang attribute |
| `examples/seo.md` | meta tags, OG tags, JSON-LD, sitemaps |
| `examples/debugging.md` | VS Code launch.json, browser breakpoints |
| `examples/migration-v2.md` | error/redirect changes, cookie path, top-level promises |

### References

| File | What it covers |
|------|----------------|
| `references/adapters-comparison.md` | detailed when-to-use-each-adapter matrix |
| `references/building-reference.md` | vite build options, preview, env vars |
| `references/routing-advanced-reference.md` | rest/optional/matchers/sort/encoding deep-dive |
| `references/layouts-advanced-reference.md` | nested/named/groups, reset, breaking out |
| `references/auth-reference.md` | sessions vs tokens, integration points, libs |
| `references/performance-reference.md` | full optimization checklist |
| `references/images-reference.md` | Vite + @sveltejs/enhanced-img reference |
| `references/a11y-reference.md` | accessibility patterns and resources |
| `references/seo-reference.md` | SEO setup, meta tags, structured data |
| `references/migration-v1-v2-reference.md` | full breaking-changes list |
| `references/migration-sapper-reference.md` | complete Sapper → SvelteKit migration |
| `references/debugging-reference.md` | IDE/debugger configs |
