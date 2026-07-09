---
name: sveltekit-overview
description: SvelteKit 概览、创建项目、项目类型、项目结构、Web 标准、Routing 技能。当用户创建新 SvelteKit 应用、选择渲染模式（SSR/SSG/SPA/MPA）、理解 +page.svelte/+page.server.js/+layout/+server.js/+error 文件约定、需要使用 Web Fetch/FormData/Stream/URL 等 Web API 时使用。
---

# SvelteKit Overview

SvelteKit is the official application framework for Svelte. It provides routing, build pipeline, server runtime, and deployment adapters so that you can ship web apps (SSR, SSG, SPA, MPA, mobile, desktop, embedded) from a single SvelteKit codebase.

## When to Use This Skill

Use this skill when you need to:

- Bootstrap a new SvelteKit project with `npx sv create`.
- Choose between rendering modes (SSR, SSG, SPA, MPA, hybrid).
- Understand the standard project layout (`src/routes`, `src/lib`, `static`, `svelte.config.js`).
- Decide between `+page.svelte` vs `+page.js` vs `+page.server.js` vs `+server.js`.
- Use Web platform APIs (`fetch`, `Request`, `Response`, `Headers`, `FormData`, `ReadableStream`, `URL`, `URLSearchParams`, `crypto`) inside SvelteKit handlers.
- Type route files with the `$types` module (`PageLoad`, `PageServerLoad`, `LayoutLoad`, `LayoutServerLoad`, `RequestHandler`, `PageProps`, `LayoutProps`).
- Build API endpoints with `+server.js` (GET / POST / PUT / PATCH / DELETE / OPTIONS / fallback).
- Configure layouts, error boundaries (`+error.svelte`), and nested routing.

## Critical: SvelteKit vs Svelte (When to Use Which)

Svelte is the UI component layer. SvelteKit is the application framework that wraps it.

| Need | Use |
|---|---|
| Render a single component, embed into an existing site | Svelte only |
| Build a full app with routing, data loading, SSR, adapters | **SvelteKit** |
| SSG / SSR / SPA / MPA / serverless | **SvelteKit** |
| Use Svelte 5 runes inside a page | Svelte inside SvelteKit |

Rule of thumb: if you have more than one "page", a backend, or want SEO, use SvelteKit.

```sh
# Pure Svelte (no routing, no SSR)
npm create vite@latest my-app -- --template svelte
# Full SvelteKit app (routing + SSR + adapters)
npx sv create my-app
```

## Critical: Creating a Project (npx sv create)

```sh
# Minimal
npx sv create my-app
cd my-app
npm install
npm run dev    # http://localhost:5173
```

Useful flags (see the `sv` CLI docs):

```sh
# Non-interactive with explicit choices
npx sv create my-app \
  --template minimal --types ts --no-add-ons --install npm

# Pick a template
npx sv create my-app --template demo   # demo | minimal | library

# Add common add-ons later
npx sv add prettier eslint vitest playwright tailwindcss
```

The CLI scaffolds: `package.json`, `svelte.config.js`, `vite.config.js`, `tsconfig.json`, `src/app.html`, `src/routes/+page.svelte`, and (depending on options) `hooks.client.js`, `hooks.server.js`, `service-worker.js`.

## Critical: Project Types Decision Matrix

Rendering settings are configured per-adapter and per-page (`export const prerender = true`, `export const ssr = false`, `export const csr = false`). Mix-and-match is the norm.

| Project type | Adapter | Config | Use when |
|---|---|---|---|
| **Default (SSR + CSR)** | any | none | Most apps. SSR first paint, CSR after navigation. |
| **Static site generation (SSG)** | `adapter-static` | `prerender = true` (root or per-page) | Docs, blogs, marketing. |
| **Single-page app (SPA)** | `adapter-static` with fallback | `ssr = false` | App shell + JSON backend, no SEO. |
| **Multi-page app (MPA)** | any | `<body data-sveltekit-reload>` + `csr = false` | Low-JS, server-driven pages. |
| **Separate backend** | `adapter-node` or serverless | skip `server` files | Backend is Go/Rust/Java/PHP. |
| **Serverless** | `adapter-vercel` / `adapter-netlify` / `adapter-cloudflare` | none | Vercel / Netlify / Cloudflare deploy. |
| **Your own server / VPS** | `adapter-node` | none | Self-hosted Node server. |
| **Container (Docker/LXC)** | `adapter-node` | none | Inside a container image. |
| **Library** | `@sveltejs/package` add-on | choose "library" template | Publish Svelte components as a package. |
| **Offline / PWA** | any | `src/service-worker.js` | Offline-capable apps. |
| **Mobile** | `adapter-static` | `bundleStrategy: 'single'` | Wrap with Tauri or Capacitor. |
| **Desktop** | `adapter-static` | `bundleStrategy: 'single'` | Wrap with Tauri / Wails / Electron. |
| **Browser extension** | `adapter-static` or community adapter | manifest + extension config | Chrome / Firefox extension. |
| **Embedded device** | any | `bundleStrategy: 'single'` | TV / microcontroller (few concurrent requests). |

## Critical: Project Structure

```tree
my-project/
├ src/
│ ├ lib/
│ │ ├ server/      # server-only lib (use $lib/server alias)
│ │ └ [your libs]  # client-safe libs (use $lib alias)
│ ├ params/        # param matchers
│ ├ routes/        # filesystem router (one folder per route)
│ ├ app.html       # HTML template (placeholders: %sveltekit.head% etc.)
│ ├ error.html     # static fallback error page
│ ├ hooks.client.js
│ ├ hooks.server.js
│ ├ service-worker.js
│ └ instrumentation.server.js
├ static/          # served verbatim (robots.txt, favicon.ico)
├ tests/           # Playwright tests (if added)
├ package.json
├ svelte.config.js
├ tsconfig.json
└ vite.config.js
```

Key rules:

- `$lib` → `src/lib` (importable from anywhere).
- `$lib/server` → `src/lib/server` (server-only — SvelteKit blocks client imports).
- `src/routes` defines the URL tree. Each folder is a route; `+page.svelte` makes it a page.
- Anything not under `src/routes` or `src/app.html` is optional.
- Prefer `import` for assets (Vite hashes them) over `static/`.

## Critical: svelte.config.js

The minimum configuration: an empty object. `svelte.config.js` exports a `SvelteConfig` object that configures `kit`, `preprocess`, `compilerOptions`, etc.

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-auto';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

/** @type {import('@sveltejs/kit').Config} */
const config = {
  preprocess: vitePreprocess(),
  kit: {
    adapter: adapter(),
    alias: {
      $components: 'src/lib/components',
      $utils: 'src/lib/utils'
    }
  }
};

export default config;
```

See `examples/svelte-config.md` for 8+ common configurations.

## Critical: Web Standards (Fetch / FormData / Streams / URL / Web Crypto)

SvelteKit "uses the platform". The browser-native APIs are available in `load` functions, `+server.js` handlers, and hooks. SvelteKit polyfills them on Node-based adapters.

| API | Where it shows up in SvelteKit |
|---|---|
| `fetch` | `load`, hooks, `+server.js`, both server and client |
| `Request` | `event.request` in hooks and `+server.js` |
| `Response` | return value of `+server.js` handlers and `fetch()` |
| `Headers` | `event.request.headers`, `Response.headers` |
| `FormData` | form submissions: `await event.request.formData()` |
| `ReadableStream` | response bodies (streaming, SSE) |
| `URL` | `event.url`, `page.url`, `beforeNavigate`/`afterNavigate` |
| `URLSearchParams` | `url.searchParams.get('foo')` |
| `crypto` | `crypto.randomUUID()`, CSPRNG, hashing |

Special note: inside `load` functions and `+server.js`, `fetch` accepts relative URLs and preserves credentials automatically. Outside those, pass cookies/headers explicitly.

## Critical: Routing (+page, +layout, +server, +error, $types)

The filesystem is the router. Route files start with `+`.

| File | Purpose |
|---|---|
| `+page.svelte` | The page UI. |
| `+page.js` | Universal `load` function (server + client). |
| `+page.server.js` | Server-only `load` and form `actions`. |
| `+layout.svelte` | Wraps pages and nested layouts. |
| `+layout.js` / `+layout.server.js` | Data load for layouts. |
| `+server.js` | API endpoint (exports HTTP verb handlers). |
| `+error.svelte` | Per-route error boundary. |
| `*.js` (other) | Ignored — colocate components or utils. |

Three rules to memorize:

1. All route files can run on the server.
2. All route files run on the client EXCEPT `+server.js`.
3. `+layout` and `+error` apply to the directory and ALL subdirectories.

### +page triad

```svelte
<!-- src/routes/blog/[slug]/+page.svelte -->
<script>
  /** @type {import('./$types').PageProps} */
  let { data } = $props();
</script>

<h1>{data.title}</h1>
```

```js
// src/routes/blog/[slug]/+page.js  -- universal load
/** @type {import('./$types').PageLoad} */
export function load({ params }) {
  return { title: params.slug };
}
```

```js
// src/routes/blog/[slug]/+page.server.js  -- server-only
import { error } from '@sveltejs/kit';
/** @type {import('./$types').PageServerLoad} */
export async function load({ params }) {
  const post = await db.posts.find(params.slug);
  if (!post) error(404, 'not found');
  return { post };
}
```

### +layout triad

Layouts wrap pages. The root `+layout.svelte` MUST render `children`:

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  let { children } = $props();
</script>
<nav><a href="/">Home</a></nav>
{@render children()}
```

### +server.js (API endpoint)

```js
// src/routes/api/random-number/+server.js
import { json, error } from '@sveltejs/kit';

/** @type {import('./$types').RequestHandler} */
export function GET({ url }) {
  const min = Number(url.searchParams.get('min') ?? 0);
  const max = Number(url.searchParams.get('max') ?? 1);
  if (max <= min) error(400, 'invalid range');
  return json({ value: min + Math.random() * (max - min) });
}
```

Handlers: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, `HEAD`, plus a special `fallback` that catches everything else.

### +error.svelte

```svelte
<!-- src/routes/blog/[slug]/+error.svelte -->
<script>
  import { page } from '$app/state';
</script>
<h1>{page.status} — {page.error.message}</h1>
```

SvelteKit walks UP the tree to find the closest `+error.svelte`. If none, the static `src/error.html` is used.

### $types

SvelteKit auto-generates `./$types.d.ts` for every route file. Use them for type safety:

| Type | Used in |
|---|---|
| `PageLoad` | `+page.js` |
| `PageServerLoad` | `+page.server.js` |
| `LayoutLoad` | `+layout.js` |
| `LayoutServerLoad` | `+layout.server.js` |
| `RequestHandler` | `+server.js` |
| `PageProps` | `+page.svelte` (`{ data, params, form }`) |
| `LayoutProps` | `+layout.svelte` (`{ data, children }`) |
| `PageData`, `LayoutData`, `ActionData` | older / narrower typing |

With VS Code + Svelte extension, types are inferred — you can skip the explicit annotations.

## Quick Fixes

- **Cannot find module '$lib'** — `svelte.config.js` ships with the `$lib` alias by default; check that your `svelte.config.js` extends the right config and that you ran `npm run dev` at least once.
- **+page.server.js data not showing** — confirm the file is named EXACTLY `+page.server.js` (with the `+`).
- **`+layout` not nesting** — the layout must `{@render children()}` somewhere in its template.
- **404 on dynamic param** — `[slug]` folder names create params; `[...rest]` is a "rest" param (greedy).
- **Hydration mismatch warning** — usually caused by `Date.now()` / `Math.random()` rendering different values on server vs client. Move them to `onMount` or `+page.server.js`.
- **`Cannot import $lib/server into client code`** — files under `src/lib/server` are server-only. Move shared code to `src/lib`.
- **Service worker not updating** — bump the version in `src/service-worker.js` (it's a build-time cache buster).
- **Body extension warning in dev** — your `app.html` should put `%sveltekit.body%` inside a `<div>`, not directly in `<body>`.
- **`Cannot read properties of undefined (reading 'request')`** — the handler is missing `event` destructuring or you forgot to declare `export async function POST(event)` / `GET({ request, url })`.
- **Types complain in JSDoc** — ensure your `tsconfig.json` `extends` `./.svelte-kit/tsconfig.json`.

## Gotchas

- `+page.js` runs on BOTH server and client. Don't put secrets or DB calls there — use `+page.server.js`.
- Data returned from `+page.server.js` is serialized with `devalue`; functions / class instances / `Map` / `Set` need explicit conversion.
- `+server.js` files are NOT wrapped by `+layout.svelte`. They emit raw responses. Use `handle` hook for cross-cutting concerns.
- `+error.svelte` is NOT used for errors thrown in `+server.js` or `handle` hook. Use `src/error.html` for that fallback.
- `csr = false` removes ALL JavaScript on the page — including the client router. Links become full reloads.
- `prerender = true` at the root layout prerenders ALL routes. Override per-page with `prerender = false`.
- `<a href="/foo">` uses the client router. Use `<a href="/foo" data-sveltekit-reload>` to force a server navigation.
- `URLSearchParams.get()` returns `null` (not `undefined`) when the key is missing.
- `event.url` in `+server.js` is a full URL; on the client `page.url` is also full; relative URLs work in `load`'s special fetch.
- Dynamic segments match across `/` boundaries ONLY if you use rest params `[...rest]`.
- Browser extensions injecting elements into `<body>` directly can break hydration — keep `%sveltekit.body%` inside a `<div>`.

## FAQ

**Q: Do I need TypeScript?**
No. `npx sv create` can scaffold a JS-only project. JSDoc types still work and provide type safety.

**Q: Can I use Svelte 5 runes in SvelteKit?**
Yes. SvelteKit 2.20+ supports Svelte 5. Use `$state`, `$derived`, `$props`, `$effect` as usual.

**Q: How do I call my own backend from a `+page.server.js`?**
Use `fetch` (the SvelteKit special one inside `load` preserves cookies and accepts relative URLs). Outside `load`, pass `cookie`/`authorization` headers explicitly.

**Q: Can `+page.js` and `+page.server.js` coexist?**
No. Use one or the other for the same route. Prefer `+page.server.js` for sensitive data.

**Q: Where do I put API keys?**
Server-only env vars in `$env/static/private` (inlined at build time) or `$env/dynamic/private` (read at runtime). Never expose private env to client code.

**Q: What's the difference between `error(404)` and `throw error(404)`?**
`error()` returns a special object — DON'T throw it. Just call `error(404, 'msg')` and SvelteKit handles the rest.

**Q: How do I get query params in a page?**
`$page.url.searchParams.get('foo')` (Svelte 4 store) or `page.url.searchParams.get('foo')` (Svelte 5 via `$app/state`).

**Q: Can I make a static-only site?**
Yes — install `adapter-static` and set `prerender = true` in the root layout (or per-page). Build output is plain HTML.

**Q: How does SvelteKit differ from Next.js / Nuxt?**
File-based router, SSR-then-CSR default, no built-in API style — you write `+server.js` for endpoints. Adapters are official and well-maintained.

## Examples & References

### Examples

| File | Description |
|---|---|
| [examples/README.md](examples/README.md) | Index of all examples |
| [examples/project-creation.md](examples/project-creation.md) | `npx sv create`, --template, --types, --add, --install |
| [examples/project-types.md](examples/project-types.md) | Each project type with adapter + config snippet |
| [examples/project-structure.md](examples/project-structure.md) | `src/`, routes, static, `package.json`, `svelte.config.js` |
| [examples/svelte-config.md](examples/svelte-config.md) | Common `svelte.config.js` setups (preprocess, kit, alias) |
| [examples/web-standards.md](examples/web-standards.md) | fetch with credentials, Request, Response, FormData, streams, URL |
| [examples/routing-pages.md](examples/routing-pages.md) | +page.svelte, +page.js, +page.server.js, params, layouts |
| [examples/routing-server.md](examples/routing-server.md) | +server.js GET/POST/PUT/DELETE, content negotiation, fallback |

### References

| File | Description |
|---|---|
| [references/README.md](references/README.md) | Index of all references |
| [references/project-types-reference.md](references/project-types-reference.md) | Full decision matrix for all project types |
| [references/project-structure-reference.md](references/project-structure-reference.md) | Every file in the standard project layout |
| [references/web-standards-reference.md](references/web-standards-reference.md) | Fetch / Request / Response / FormData / Streams / URL / Web Crypto in SvelteKit |
| [references/routing-reference.md](references/routing-reference.md) | +page, +layout, +server, +error, $types reference |