# Performance

SvelteKit ships with code-splitting, asset preloading, file hashing, request coalescing, parallel loading, data inlining, conservative invalidation, link preloading. To go further:

## 1. Diagnose first

- **PageSpeed Insights**: https://pagespeed.web.dev/
- **WebPageTest**: https://www.webpagetest.org/ (advanced)
- **Chrome DevTools → Lighthouse / Network / Performance**
- Test in preview mode (`npm run build && npm run preview`), not dev

## 2. Code-split with dynamic imports

```svelte
<script>
  import { browser } from '$app/environment';
  let HeavyComponent;
  
  $effect(() => {
    if (browser && someCondition) {
      import('$lib/HeavyComponent.svelte').then(m => HeavyComponent = m.default);
    }
  });
</script>
```

Or use `await import(...)` inside an event handler:

```svelte
<button onclick={async () => {
  const { heavy } = await import('$lib/heavy');
  heavy();
}}>Run</button>
```

## 3. Identify large packages

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

`npm run build && open stats.html`.

## 4. Avoid client → server → backend waterfalls

```js
// BAD: universal load calls API, then uses response to call another API
export async function load({ fetch }) {
  const user = await fetch('/api/me').then(r => r.json());
  const items = await fetch(`/api/users/${user.id}/items`).then(r => r.json());
  return { user, items };
}

// GOOD: server load does both from one DB query
// +page.server.js
export async function load({ locals }) {
  const [user, items] = await Promise.all([
    db.getUser(locals.userId),
    db.getItems(locals.userId)
  ]);
  return { user, items };
}
```

## 5. Use parallel queries for multiple URLs

```js
// +page.server.js
export async function load({ fetch }) {
  const [a, b, c] = await Promise.all([
    fetch(url1).then(r => r.json()),
    fetch(url2).then(r => r.json()),
    fetch(url3).then(r => r.json())
  ]);
  return { a, b, c };
}
```

## 6. Stream non-essential data

Return promises from `load` for slow data:

```js
// +page.server.js
export async function load({ fetch }) {
  return {
    user: fetch('/api/me').then(r => r.json()),  // streams
    settings: await fetch('/api/settings').then(r => r.json())  // blocks
  };
}
```

```svelte
{#await data.user}
  <p>Loading user...</p>
{:then user}
  <User {user} />
{/await}
```

## 7. Preload critical fonts

```js
// src/hooks.server.js
export async function handle({ event, resolve }) {
  return resolve(event, {
    preload: ({ type, path }) => type === 'js' || path.endsWith('.woff2')
  });
}
```

Subset fonts with `pyftsubset` or Glyphhanger to reduce size.

## 8. Lazy-load videos

```html
<video preload="none" poster="thumb.jpg" controls>
  <source src="video.webm" type="video/webm" />
</video>
```

Strip audio from muted clips via FFmpeg. Convert to `.webm` for size savings.

## 9. Push third-party scripts to a worker

Use Partytown to run analytics / chat widgets off the main thread:

```sh
npm i -D @builder.io/partytown
```

```js
// src/routes/+layout.svelte
<script>
  import { partytownSnippet } from '@builder.io/partytown/integration';
  let { children } = $props();
</script>

<svelte:head>
  {@html partytownSnippet()}
</svelte:head>

{@render children()}
```

```html
<script type="text/partytown" src="https://example.com/analytics.js"></script>
```

## 10. Server-side analytics

Replace JS-based analytics with platform-native:

- Cloudflare Web Analytics (no JS)
- Netlify Analytics (server-side)
- Vercel Analytics (edge, lightweight)

No client-side bundle, no performance hit.

## 11. SPA mode penalty

Avoid SPA mode (`fallback: '200.html'` + `ssr = false`) unless absolutely necessary. It causes 3+ round trips before first paint:
1. Empty HTML
2. JS bundle
3. Data load

Prerender instead when possible.

## 12. Edge deployment

Use edge adapters (Cloudflare, Netlify Edge, Vercel Edge) to reduce latency for global users. Verify your code works without Node APIs (`fs`, etc).

## See also

- [Performance reference](../references/performance-reference.md)
