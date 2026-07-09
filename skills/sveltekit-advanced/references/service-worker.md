# Service Worker Reference

## Setup

Place `src/service-worker.js` (or `src/service-worker/index.js`). SvelteKit auto-registers it.

Disable auto-register: `kit.serviceWorker.register = false` in config.

## $service-worker module

```ts
import {
  base,         // string — deployment base path (computed from location.pathname)
  build,        // string[] — Vite-built app files (empty in dev)
  files,        // string[] — static directory files
  prerendered,  // string[] — prerendered pages (empty in dev)
  version       // string — config.kit.version.name (use for cache name)
} from '$service-worker';
```

`base` adapts if deployed to a subdirectory. Service workers cannot use `config.kit.paths.assets` (the `assets` config).

## Triple-slash directives

Required at the top of `src/service-worker.js`:

```js
/// <reference no-default-lib="true"/>
/// <reference lib="esnext" />
/// <reference lib="webworker" />
/// <reference types="@sveltejs/kit" />
/// <reference types="../.svelte-kit/ambient.d.ts" />
```

## Type setup

```js
const self = /** @type {ServiceWorkerGlobalScope} */ (
  /** @type {unknown} */ (globalThis.self)
);
```

## Cache strategies

### Cache-first (app shell)

```js
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  if (build.includes(url.pathname) || files.includes(url.pathname)) {
    event.respondWith(caches.match(event.request));
  }
});
```

### Network-first with cache fallback

```js
async function networkFirst(request) {
  const cache = await caches.open(CACHE);
  try {
    const response = await fetch(request);
    if (response.status === 200 &&
        !response.headers.get('cache-control')?.includes('no-store')) {
      cache.put(request, response.clone());
    }
    return response;
  } catch (err) {
    return (await cache.match(request)) ?? Response.error();
  }
}
```

### Stale-while-revalidate

```js
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE);
  const cached = await cache.match(request);
  const fetched = fetch(request).then((response) => {
    if (response.ok) cache.put(request, response.clone());
    return response;
  });
  return cached ?? fetched;
}
```

## Install / activate

```js
const CACHE = `cache-${version}`;
const ASSETS = [...build, ...files];

self.addEventListener('install', (event) => {
  event.waitUntil(caches.open(CACHE).then((c) => c.addAll(ASSETS)));
});

self.addEventListener('activate', (event) => {
  event.waitUntil((async () => {
    for (const key of await caches.keys()) {
      if (key !== CACHE) await caches.delete(key);
    }
  })());
});
```

## Skip live queries

Live query responses have `Cache-Control: no-store`. Never cache them — the stream stays open after the page closes.

```js
if (response.status === 200 &&
    !response.headers.get('cache-control')?.includes('no-store')) {
  cache.put(request, response.clone());
}
```

## Manual registration

```js
// svelte.config.js
export default { kit: { serviceWorker: { register: false } } };

// client code
if ('serviceWorker' in navigator) {
  addEventListener('load', () => {
    navigator.serviceWorker.register('./path/to/service-worker.js', {
      type: import.meta.env.DEV ? 'module' : 'classic'
    });
  });
}
```

The service worker file is bundled for production but NOT during development.

## Custom file filtering

```js
// svelte.config.js
serviceWorker: {
  files: (filepath) => !filepath.includes('large-')
}
```

## Update flow

New `version` (from `config.kit.version.name`) → new cache name → old caches deleted in `activate`.

To force update on next page load:

```js
self.addEventListener('install', () => self.skipWaiting());
self.addEventListener('activate', (event) => {
  event.waitUntil(self.clients.claim());
});
```

## Caching limits

Browsers evict caches when full. Avoid caching:
- Large video files.
- Rarely-changing but huge assets.

`build`/`prerendered` are EMPTY in dev — test caching in production build.

## Workbox alternative

SvelteKit's built-in works for most cases. For Workbox users: see [Vite PWA plugin](https://vite-pwa-org.netlify.app/frameworks/sveltekit.html).

## Push notifications

```js
self.addEventListener('push', (event) => {
  const data = event.data?.json();
  event.waitUntil(
    self.registration.showNotification(data.title, { body: data.body })
  );
});
```

## References

- [MDN: Using Service Workers](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers)
- [web.dev: Workbox](https://web.dev/learn/pwa/workbox)