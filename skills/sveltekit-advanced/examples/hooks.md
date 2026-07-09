# Hooks Examples

`src/hooks.server.js`, `src/hooks.client.js`, and `src/hooks.js`.

## Server hooks

### 1. Basic handle — short-circuit

```js
// src/hooks.server.js
export async function handle({ event, resolve }) {
  if (event.url.pathname.startsWith('/custom')) {
    return new Response('custom response');
  }
  return resolve(event);
}
```

### 2. Populate locals (auth)

```js
// src/hooks.server.js
export async function handle({ event, resolve }) {
  event.locals.user = await getUser(event.cookies.get('sessionid'));

  const response = await resolve(event);

  response.headers.set('x-custom-header', 'potato');
  return response;
}
```

Declare locals in `src/app.d.ts`:

```ts
declare global {
  namespace App {
    interface Locals { user: User | null; }
  }
}
```

### 3. transformPageChunk / preload / filterSerializedResponseHeaders

```js
export async function handle({ event, resolve }) {
  return resolve(event, {
    transformPageChunk: ({ html }) => html.replace('old', 'new'),
    filterSerializedResponseHeaders: (name) => name.startsWith('x-'),
    preload: ({ type, path }) => type === 'js' || path.includes('/important/')
  });
}
```

### 4. Sequence multiple handle functions

```js
import { sequence } from '@sveltejs/kit/hooks';

async function auth({ event, resolve }) {
  event.locals.user = await getUser(event.cookies.get('session'));
  return resolve(event);
}

async function logger({ event, resolve }) {
  const start = Date.now();
  const response = await resolve(event);
  console.log(`${event.url.pathname} ${Date.now() - start}ms`);
  return response;
}

export const handle = sequence(auth, logger);
```

### 5. handleFetch — rewrite external URL to internal

```js
export async function handleFetch({ request, fetch }) {
  if (request.url.startsWith('https://api.yourapp.com/')) {
    request = new Request(
      request.url.replace('https://api.yourapp.com/', 'http://localhost:9999/'),
      request
    );
  }
  return fetch(request);
}
```

### 6. handleFetch — sibling subdomain cookie

```js
export async function handleFetch({ event, request, fetch }) {
  if (request.url.startsWith('https://api.my-domain.com/')) {
    request.headers.set('cookie', event.request.headers.get('cookie'));
  }
  return fetch(request);
}
```

### 7. handleValidationError — customize 400

```js
export function handleValidationError({ issues }) {
  return { message: 'Nice try, hacker!' };
}
```

## Shared hooks (server + client)

### 8. handleError with Sentry (server)

```js
// src/hooks.server.js
import * as Sentry from '@sentry/sveltekit';

Sentry.init({/* ... */});

/** @type {import('@sveltejs/kit').HandleServerError} */
export async function handleError({ error, event, status, message }) {
  const errorId = crypto.randomUUID();
  Sentry.captureException(error, { extra: { event, errorId, status } });
  return { message: 'Whoops!', errorId };
}
```

### 9. handleError with Sentry (client)

```js
// src/hooks.client.js
import * as Sentry from '@sentry/sveltekit';

Sentry.init({/* ... */});

/** @type {import('@sveltejs/kit').HandleClientError} */
export async function handleError({ error, event, status, message }) {
  const errorId = crypto.randomUUID();
  Sentry.captureException(error, { extra: { event, errorId, status } });
  return { message: 'Whoops!', errorId };
}
```

Note: client uses `HandleClientError` and `event` is `NavigationEvent`, not `RequestEvent`.

### 10. init — startup work

```js
// src/hooks.server.js
import * as db from '$lib/server/database';

/** @type {import('@sveltejs/kit').ServerInit} */
export async function init() {
  await db.connect();
}
```

### 11. init (client) — be cautious

```js
// src/hooks.client.js
export async function init() {
  // Asynchronous work here delays hydration. Keep it minimal.
  console.log('client init');
}
```

## Universal hooks (src/hooks.js)

### 12. reroute — i18n

```js
// src/hooks.js
const translated = {
  '/en/about': '/en/about',
  '/de/ueber-uns': '/de/about',
  '/fr/a-propos': '/fr/about'
};

export function reroute({ url }) {
  if (url.pathname in translated) return translated[url.pathname];
}
```

`lang` parameter is correctly derived.

### 13. Async reroute (since 2.18)

```js
export async function reroute({ url, fetch }) {
  if (url.pathname === '/api/reroute') return;
  const api = new URL('/api/reroute', url);
  api.searchParams.set('pathname', url.pathname);
  const result = await fetch(api).then((r) => r.json());
  return result.pathname;
}
```

`reroute` is cached per URL on the client; must be pure/idempotent.

### 14. transport — custom types

```js
// src/hooks.js
import { Vector } from '$lib/math';

export const transport = {
  Vector: {
    encode: (value) => value instanceof Vector && [value.x, value.y],
    decode: ([x, y]) => new Vector(x, y)
  }
};
```

Allows `Vector` instances to cross the server/client boundary in `load` results.

### 15. Skip prerender code

```js
export async function handle({ event, resolve }) {
  if (!building) {
    // setup that shouldn't run during prerender
  }
  return resolve(event);
}
```

`building` from `$app/environment` is `true` during `vite build` and prerender.

### 16. Resolve options precedence in sequence

```js
export const handle = sequence(first, second);
```

- `transformPageChunk`: applied in REVERSE order (second then first), merged
- `preload`: applied FORWARD; first option wins
- `filterSerializedResponseHeaders`: same as preload (first wins)