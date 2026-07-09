# Performance reference

Comprehensive checklist for optimizing a SvelteKit app. SvelteKit already handles most of this — focus on what's left.

## What SvelteKit does for you

- Code-splitting per route
- Asset preloading (modulepreload)
- File hashing (long-cacheable assets)
- Request coalescing (parallel server load → one HTTP request)
- Parallel loading (universal loads run concurrently)
- Data inlining (server fetches replayed in browser)
- Conservative invalidation (loads only re-run when needed)
- Per-route prerendering
- Link preloading (hover/tap)

## Diagnose

### Tools

- **PageSpeed Insights** (https://pagespeed.web.dev/) — quick overall score
- **WebPageTest** (https://www.webpagetest.org/) — filmstrip, waterfall, multi-run
- **Chrome DevTools** — Network tab, Performance tab, Lighthouse
- **Edge browser DevTools** — same as Chrome
- **Firefox Profiler** — flame charts for JS
- **bundle.js.org** — visualize bundle composition

### Test in production mode

`vite dev` is dramatically slower than production. Always test `npm run build && npm run preview`, or use the platform CLI.

### Server-side timing

OpenTelemetry via `src/instrumentation.server.ts` + `kit.experimental.tracing.server = true`. See SvelteKit docs for setup with Jaeger.

Add Server-Timing headers via `handle`:

```js
return resolve(event, {
  transformPageChunk: ({ html }) => {
    // ...
  }
}).then(r => {
  // modify response headers
});
```

Or directly with the Response object from `resolve`.

## Asset optimization

### Images

- Use `@sveltejs/enhanced-img` for static images
- Use `@unpic/svelte` or CDN-native for dynamic images
- Provide 2x source for HiDPI displays
- Set `width`/`height` to prevent CLS
- LCP images: `fetchpriority="high"`, no `loading="lazy"`
- Modern formats: AVIF → WebP → fallback JPEG

### Videos

- Compress with Handbrake
- Convert to `.webm` or `.mp4`
- `preload="none"` for below-fold videos
- Strip audio from muted clips (FFmpeg)

### Fonts

- Subset to characters you actually use (Glyphhanger, pyftsubset)
- Use `font-display: swap` or `optional`
- Preload critical fonts via `handle`'s `preload` option
- Self-host for privacy + to remove third-party connection

### CSS

- Tailwind or UnoCSS for atomic CSS (tree-shakes unused)
- Critical CSS inlined by SvelteKit automatically
- Avoid `@import` in CSS (slow)
- Use CSS containment (`contain: layout`) for layout isolation

## Code size

### Upgrade Svelte

Svelte 5 is smaller than 4, 4 smaller than 3. Always use the latest.

### Find heavy packages

```sh
npm i -D rollup-plugin-visualizer
```

```js
// vite.config.js
import { visualizer } from 'rollup-plugin-visualizer';
export default {
  plugins: [sveltekit(), visualizer({ filename: 'stats.html', gzipSize: true })]
};
```

`npm run build && open stats.html` — sorted by size.

### Manual inspection

`vite.config.js`:

```js
export default {
  build: { minify: false }   // readable output (revert before deploying)
};
```

### Selective loading

Static imports are always bundled. Use dynamic for conditional code:

```svelte
<script>
  let Heavy;
  $effect(() => {
    if (visible) {
      import('$lib/Heavy.svelte').then(m => Heavy = m.default);
    }
  });
</script>
```

### External scripts

Move analytics / chat widgets to:
- Server-side platforms (Cloudflare Web Analytics, Netlify Analytics)
- Partytown (web worker)

## Avoiding waterfalls

### Server-side

Always prefer **server `load`** for backend calls (no client → server → backend chain):

```js
// +page.server.js — calls DB directly
export async function load({ locals }) {
  const user = await db.getUser(locals.userId);
  const items = await db.getItems(locals.userId);
  return { user, items };
}
```

Universal `load` runs on both server AND client. If the data is server-only (DB queries with secrets), use server `load`.

### Multiple parallel requests

```js
const [a, b] = await Promise.all([fetch(url1), fetch(url2)]);
```

Or use a single DB query with joins.

### Streaming

Return promises from `load` for non-critical data:

```js
return {
  user: await fetch('/api/me'),     // blocks
  recommendations: fetch('/api/recs')  // streams
};
```

## Hosting

- **Co-locate** frontend and backend (same region)
- **Edge deployment** for global users (Cloudflare, Vercel Edge, Netlify Edge)
- **HTTP/2+** required (Vite's many small files rely on multiplexing)
- **CDN** for static assets — most adapters do this automatically
- **Compression** — at reverse proxy (nginx, Caddy) or pre-compressed (adapter option)

## Caching headers

```js
// src/hooks.server.js
export const handle = async ({ event, resolve }) => {
  const response = await resolve(event);
  
  if (event.url.pathname.startsWith('/_app/')) {
    response.headers.set('Cache-Control', 'public, max-age=31536000, immutable');
  }
  
  return response;
};
```

`_app/` assets are hashed → safe to cache forever.

## Avoiding common perf killers

- **SPA mode** (`fallback: '200.html'` + `ssr = false`) — extra round trips
- **Massive client-side state** — Svelte 5 runes are efficient, but huge stores still hurt
- **Synchronous `localStorage`/`sessionStorage`** in initial render — wrap in `onMount`
- **Large SVG sprites** — use Iconify via CSS or individual components
- **Lazy hydration** — not built-in, but consider Svelte's `<svelte:component>` and conditional rendering

## See also

- [examples/performance.md](../examples/performance.md)
- [examples/images.md](../examples/images.md)
- [images-reference.md](images-reference.md)
