# Service Worker Examples

Offline support and asset precaching via `src/service-worker.js`.

## 1. Basic offline-first service worker

```js
// src/service-worker.js
/// <reference no-default-lib="true"/>
/// <reference lib="esnext" />
/// <reference lib="webworker" />
/// <reference types="@sveltejs/kit" />
/// <reference types="../.svelte-kit/ambient.d.ts" />

import { build, files, version } from '$service-worker';

const self = /** @type {ServiceWorkerGlobalScope} */ (/** @type {unknown} */ (globalThis.self));
const CACHE = `cache-${version}`;

const ASSETS = [...build, ...files];

self.addEventListener('install', (event) => {
  async function addFilesToCache() {
    const cache = await caches.open(CACHE);
    await cache.addAll(ASSETS);
  }
  event.waitUntil(addFilesToCache());
});

self.addEventListener('activate', (event) => {
  async function deleteOldCaches() {
    for (const key of await caches.keys()) {
      if (key !== CACHE) await caches.delete(key);
    }
  }
  event.waitUntil(deleteOldCaches());
});

self.addEventListener('fetch', (event) => {
  if (event.request.method !== 'GET') return;

  async function respond() {
    const url = new URL(event.request.url);
    const cache = await caches.open(CACHE);

    if (ASSETS.includes(url.pathname)) {
      const response = await cache.match(url.pathname);
      if (response) return response;
    }

    try {
      const response = await fetch(event.request);
      if (!(response instanceof Response)) {
        throw new Error('invalid response from fetch');
      }

      if (response.status === 200 && !response.headers.get('cache-control')?.includes('no-store')) {
        cache.put(event.request, response.clone());
      }
      return response;
    } catch (err) {
      const response = await cache.match(event.request);
      if (response) return response;
      throw err;
    }
  }

  event.respondWith(respond());
});
```

`build` is empty in dev — test in production build.

## 2. Cache-first for app shell, network-first for data

```js
self.addEventListener('fetch', (event) => {
  if (event.request.method !== 'GET') return;
  const url = new URL(event.request.url);

  // App shell — cache-first
  if (build.includes(url.pathname) || files.includes(url.pathname)) {
    event.respondWith(caches.match(event.request));
    return;
  }

  // API calls — network-first
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(event.request));
    return;
  }
});

async function networkFirst(request) {
  const cache = await caches.open(`api-${version}`);
  try {
    const response = await fetch(request);
    if (response.ok) cache.put(request, response.clone());
    return response;
  } catch {
    return (await cache.match(request)) ?? Response.error();
  }
}
```

## 3. Disable auto-registration

```js
// svelte.config.js
export default {
  kit: {
    serviceWorker: { register: false }
  }
};
```

Then manually:

```js
// In your client code
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/service-worker.js', {
    type: import.meta.env.DEV ? 'module' : 'classic'
  });
}
```

## 4. Exclude specific files from precache

```js
// svelte.config.js
export default {
  kit: {
    serviceWorker: {
      files: (filepath) => !filepath.endsWith('large-video.mp4')
    }
  }
};
```

## 5. Use $service-worker

```js
import { base, build, files, prerendered, version } from '$service-worker';

console.log(base);           // '/myapp' if deployed to subdir
console.log(build);          // ['/_app/immutable/...', ...]
console.log(files);          // ['/favicon.png', '/robots.txt', ...]
console.log(prerendered);    // ['/about', '/blog/post-1', ...]
console.log(version);        // unique per build — use for cache names
```

`base` is calculated from `location.pathname`, so it works even deployed to a subdirectory.

## 6. Cache prerendered pages

```js
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE).then((cache) =>
      cache.addAll([...build, ...files, ...prerendered])
    )
  );
});
```

## 7. Skip live query streams

Live query responses have `Cache-Control: no-store`. Don't cache them:

```js
if (response.status === 200 && !response.headers.get('cache-control')?.includes('no-store')) {
  cache.put(event.request, response.clone());
}
```

Caching a live stream keeps it open in the SW after the page closes.

## 8. Push notification setup

```js
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? { title: 'Notification', body: '' };
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon.png'
    })
  );
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  event.waitUntil(self.clients.openWindow('/'));
});
```

## 9. Background sync

```js
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-messages') {
    event.waitUntil(sendQueuedMessages());
  }
});
```

## 10. Force update

```js
self.addEventListener('install', () => {
  self.skipWaiting();  // activate new SW immediately
});

self.addEventListener('activate', (event) => {
  event.waitUntil(self.clients.claim());  // take control of open clients
});
```

## 11. Stale-while-revalidate

```js
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE);
  const cached = await cache.match(request);

  const networkPromise = fetch(request).then((response) => {
    if (response.ok) cache.put(request, response.clone());
    return response;
  });

  return cached ?? networkPromise;
}
```

## 12. Cache size limit

Browsers may evict large caches. Don't precache huge media files.