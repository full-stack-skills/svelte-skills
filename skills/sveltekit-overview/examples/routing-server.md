# Routing — Server Endpoints (+server.js) Examples

`+server.js` files export HTTP verb handlers that return a `Response`. They are NEVER wrapped by layouts and never run in the browser.

## 1. Simple GET handler

```js
// src/routes/api/random-number/+server.js
import { error } from '@sveltejs/kit';

/** @type {import('./$types').RequestHandler} */
export function GET({ url }) {
  const min = Number(url.searchParams.get('min') ?? 0);
  const max = Number(url.searchParams.get('max') ?? 1);
  if (max <= min) error(400, 'invalid range');
  return new Response(String(min + Math.random() * (max - min)));
}
```

## 2. POST handler with JSON body

```js
// src/routes/api/add/+server.js
import { json } from '@sveltejs/kit';

/** @type {import('./$types').RequestHandler} */
export async function POST({ request }) {
  const { a, b } = await request.json();
  return json(a + b);
}
```

## 3. POST with FormData

```js
// src/routes/api/signup/+server.js
import { json } from '@sveltejs/kit';

/** @type {import('./$types').RequestHandler} */
export async function POST({ request }) {
  const data = await request.formData();
  const email = String(data.get('email') ?? '');
  return json({ ok: true, email });
}
```

## 4. PUT / PATCH / DELETE handlers

```js
// src/routes/api/posts/[id]/+server.js
import { json, error } from '@sveltejs/kit';
import { db } from '$lib/server/db';

/** @type {import('./$types').RequestHandler} */
export async function PUT({ params, request }) {
  const body = await request.json();
  const post = await db.posts.update(params.id, body);
  if (!post) error(404);
  return json(post);
}

/** @type {import('./$types').RequestHandler} */
export async function DELETE({ params }) {
  await db.posts.delete(params.id);
  return new Response(null, { status: 204 });
}
```

## 5. Returning various response types

```js
/** @type {import('./$types').RequestHandler} */
export function GET() {
  return new Response('plain text', { headers: { 'content-type': 'text/plain' } });
}

// JSON
return json({ hello: 'world' });

// Redirect
import { redirect } from '@sveltejs/kit';
redirect(302, '/login');

// Throw 4xx / 5xx
import { error } from '@sveltejs/kit';
error(404, 'Not found');
error(401, 'Unauthorized');
```

## 6. Content negotiation (page OR API on the same route)

`+server.js` and `+page.svelte` can coexist in the same route folder.

```tree
src/routes/blog/[slug]/
├ +page.svelte         # browser visit -> HTML
├ +page.server.js
└ +server.js           # API client -> JSON
```

SvelteKit applies these rules:

- `PUT` / `PATCH` / `DELETE` / `OPTIONS` -> always `+server.js`.
- `GET` / `POST` / `HEAD` -> `+server.js` unless `Accept: text/html` prioritizes HTML.
- `GET` responses include `Vary: Accept` so caches split HTML and JSON.

```js
// src/routes/blog/[slug]/+server.js
import { json } from '@sveltejs/kit';

/** @type {import('./$types').RequestHandler} */
export async function GET({ params }) {
  const post = await getPost(params.slug);
  return json(post);
}
```

```html
<!-- Visit /blog/hello-world in a browser: HTML page -->
<!-- Visit /blog/hello-world with curl -H "accept: application/json": JSON -->
```

## 7. Streaming response

```js
/** @type {import('./$types').RequestHandler} */
export function GET() {
  const stream = new ReadableStream({
    start(controller) {
      const enc = new TextEncoder();
      controller.enqueue(enc.encode('hello\n'));
      setTimeout(() => {
        controller.enqueue(enc.encode('world\n'));
        controller.close();
      }, 500);
    }
  });
  return new Response(stream, { headers: { 'content-type': 'text/plain' } });
}
```

## 8. Reading cookies / setting cookies

```js
// src/routes/api/me/+server.js
import { json } from '@sveltejs/kit';

/** @type {import('./$types').RequestHandler} */
export function GET({ cookies }) {
  const session = cookies.get('session');
  return json({ session });

  // to set a cookie:
  // cookies.set('session', 'abc', { path: '/', httpOnly: true, secure: true });
  // to delete:
  // cookies.delete('session', { path: '/' });
}
```

## 9. Setting CORS headers

```js
/** @type {import('./$types').RequestHandler} */
export function OPTIONS() {
  return new Response(null, {
    headers: {
      'access-control-allow-origin': '*',
      'access-control-allow-methods': 'GET, POST, OPTIONS',
      'access-control-allow-headers': 'content-type'
    }
  });
}
```

Note: Vite injects CORS headers in dev; you must set them yourself in production.

## 10. Fallback handler

```js
// src/routes/api/echo/+server.js
import { json, text } from '@sveltejs/kit';

/** @type {import('./$types').RequestHandler} */
export async function POST({ request }) {
  const body = await request.json();
  return json(body);
}

/** @type {import('./$types').RequestHandler} */
export async function fallback({ request }) {
  return text(`I caught your ${request.method} request!`);
}
```

`fallback` handles any HTTP method that doesn't have its own export (e.g. `MOVE`, `PROPFIND`). `HEAD` falls back to `GET` if present.

## 11. HEAD from GET (automatic)

```js
/** @type {import('./$types').RequestHandler} */
export function GET() {
  return new Response('Hello', {
    headers: { 'content-length': '5' }
  });
}
```

A `HEAD` request will return the same headers, including `content-length`, with no body.

## 12. Throw redirect from an endpoint

```js
import { redirect } from '@sveltejs/kit';

/** @type {import('./$types').RequestHandler} */
export function GET() {
  redirect(302, 'https://example.com');
}
```

Throw `redirect()` (don't `return`) — it throws a special object that SvelteKit catches and converts to an HTTP redirect.

## 13. Reading the body as bytes

```js
/** @type {import('./$types').RequestHandler} */
export async function POST({ request }) {
  const buf = await request.arrayBuffer();
  return new Response(buf.byteLength, { headers: { 'content-type': 'text/plain' } });
}
```

## 14. Logging the request URL

```js
/** @type {import('./$types').RequestHandler} */
export function GET({ url, request }) {
  console.log('GET', url.pathname, request.headers.get('user-agent'));
  return new Response('ok');
}
```