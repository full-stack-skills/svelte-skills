# Project Types Reference

SvelteKit offers configurable rendering. The project type is determined by the **adapter** you install and the **page options** you set (`prerender`, `ssr`, `csr`). Mix-and-match is the norm — different routes can use different rendering modes.

## Decision Matrix

| Type | Adapter | Config | Best for |
|---|---|---|---|
| Default (SSR+CSR) | `adapter-auto` (or any) | none | Most apps |
| Static site generation | `adapter-static` | `prerender = true` | Docs, blogs, marketing |
| Single-page app | `adapter-static` | `fallback: 'index.html'` + `ssr = false` | App shell + JSON API |
| Multi-page app | any | `<body data-sveltekit-reload>` + `csr = false` | Server-driven, low-JS |
| Separate backend | `adapter-node` or serverless | skip `+server`/`+page.server.js` | Backend in Go/Rust/Java/PHP |
| Serverless | `adapter-vercel` / `-netlify` / `-cloudflare` | none | Vercel / Netlify / Cloudflare |
| Self-hosted | `adapter-node` | `PORT=3000` | VPS, your own server |
| Container | `adapter-node` | Dockerfile | Docker / LXC / k8s |
| Library | `@sveltejs/package` | `--template library` | Publish Svelte components |
| Offline / PWA | any | `src/service-worker.js` | Offline-capable apps |
| Mobile | `adapter-static` | `bundleStrategy: 'single'` | Tauri / Capacitor |
| Desktop | `adapter-static` | `bundleStrategy: 'single'` | Tauri / Wails / Electron |
| Browser extension | `adapter-static` or community | extension manifest | Chrome / Firefox extension |
| Embedded device | any | `bundleStrategy: 'single'` | TV / microcontroller |

## Per-route page options

These are exported from `+page.js` / `+layout.js` (or their `.server.js` equivalents):

```js
// src/routes/about/+page.js
export const prerender = true;   // build-time static HTML
export const ssr = true;         // server-render at request time
export const csr = true;         // hydrate in the browser
```

- `prerender = true` -> at build time SvelteKit visits the route and writes static HTML.
- `prerender = 'auto'` -> SvelteKit prerenders if no `load` function depends on the request.
- `ssr = false` -> route is skipped during server rendering (only `csr` produces HTML).
- `csr = false` -> no JavaScript is sent or executed on the page.

A useful recipe for marketing + dashboard hybrid:

```js
// src/routes/+layout.js  -- marketing pages prerendered
export const prerender = true;
```

```js
// src/routes/dashboard/+layout.js  -- dashboard is SSR-only, no JS
export const prerender = false;
export const ssr = true;
export const csr = true;
```

## Adapters

| Adapter | Output | Notes |
|---|---|---|
| `adapter-auto` | Detects platform | Zero-config default |
| `adapter-static` | Static files | SSG and SPA |
| `adapter-node` | Node server | Self-hosted, Docker |
| `adapter-vercel` | Vercel functions | Edge or Node runtime |
| `adapter-netlify` | Netlify functions | Edge or Node |
| `adapter-cloudflare` | Cloudflare Workers | Edge |
| `@sveltejs/package` | npm package | Library mode |

Community adapters: <https://sveltesociety.dev/packages?category=sveltekit-adapters>

## When to choose what

- **Default** when you want SSR for SEO and CSR for snappy navigation.
- **SSG** when content rarely changes and you can serve plain HTML from a CDN.
- **SPA** when you have an existing JSON API and no SEO requirements.
- **Serverless** when you want zero ops and instant scale.
- **Self-hosted / Container** when you control the runtime or have compliance constraints.
- **Mobile / Desktop** when wrapping the same web app for native distribution.
- **Library** when publishing Svelte components for others to consume.

## Bundle strategies

For platforms that limit concurrent connections (mobile, embedded, slow networks):

```js
// svelte.config.js
export default {
  kit: { /* ... */ },
  output: { bundleStrategy: 'single' }
};
```

This bundles the entire app into a single JS file, reducing request count at the cost of a larger initial payload.

## Combining with separate backends

If you have a separate API server, two common patterns:

1. **Two deploys** — SvelteKit on Vercel, backend on Fly.io. Use `fetch('https://api.example.com/...')` from `+page.server.js` (or pass cookies for auth).
2. **SPA served by backend** — backend serves SvelteKit's `build/` directory plus a `200.html` fallback for client routes. Simpler ops but worse SEO.

In either case, ignore the parts of the SvelteKit docs about `+server.js` / form actions / `$lib/server`.