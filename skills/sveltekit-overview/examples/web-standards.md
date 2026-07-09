# Web Standards Examples

SvelteKit "uses the platform". These snippets show the Fetch, FormData, Stream, URL, and Web Crypto APIs in SvelteKit contexts.

## 1. fetch in a `load` function (relative URLs, credentials preserved)

```js
// src/routes/+page.js
/** @type {import('./$types').PageLoad} */
export async function load({ fetch }) {
  const res = await fetch('/api/items');   // relative URL works
  const items = await res.json();
  return { items };
}
```

## 2. fetch in `+page.server.js` (DB-backed)

```js
// src/routes/+page.server.js
/** @type {import('./$types').PageServerLoad} */
export async function load({ fetch, params }) {
  const res = await fetch(`https://api.example.com/posts/${params.slug}`);
  if (!res.ok) return { post: null };
  return { post: await res.json() };
}
```

## 3. fetch with explicit credentials (outside load)

```js
// outside load: cookies do NOT auto-forward
const res = await fetch('https://api.example.com/me', {
  headers: { cookie: event.request.headers.get('cookie') ?? '' }
});
```

## 4. Reading Request headers in `+server.js`

```js
// src/routes/what-is-my-user-agent/+server.js
import { json } from '@sveltejs/kit';

/** @type {import('./$types').RequestHandler} */
export function GET({ request }) {
  console.log(...request.headers);
  return json(
    { userAgent: request.headers.get('user-agent') },
    { headers: { 'x-custom-header': 'potato' } }
  );
}
```

## 5. Reading FormData from a POST

```js
// src/routes/hello/+server.js
import { json } from '@sveltejs/kit';

/** @type {import('./$types').RequestHandler} */
export async function POST({ request }) {
  const body = await request.formData();
  console.log([...body]);   // [[key, value], ...]
  return json({ name: body.get('name') ?? 'world' });
}
```

```html
<form method="POST" action="/hello">
  <input name="name" />
  <button>Send</button>
</form>
```

## 6. Reading JSON from a POST

```js
/** @type {import('./$types').RequestHandler} */
export async function POST({ request }) {
  const data = await request.json();
  return json({ received: data });
}
```

## 7. Returning a Response with a custom status

```js
/** @type {import('./$types').RequestHandler} */
export function GET() {
  return new Response('Created', { status: 201 });
}
```

## 8. Setting response headers (cache control, CORS)

```js
import { json } from '@sveltejs/kit';

/** @type {import('./$types').RequestHandler} */
export function GET() {
  return json(
    { ok: true },
    {
      headers: {
        'cache-control': 'public, max-age=3600',
        'access-control-allow-origin': '*'
      }
    }
  );
}
```

## 9. Streaming a large response

```js
/** @type {import('./$types').RequestHandler} */
export function GET() {
  const stream = new ReadableStream({
    start(controller) {
      controller.enqueue(new TextEncoder().encode('chunk 1\n'));
      controller.enqueue(new TextEncoder().encode('chunk 2\n'));
      controller.close();
    }
  });
  return new Response(stream, {
    headers: { 'content-type': 'text/plain' }
  });
}
```

## 10. Server-Sent Events

```js
/** @type {import('./$types').RequestHandler} */
export function GET() {
  let id = 0;
  const stream = new ReadableStream({
    start(controller) {
      const interval = setInterval(() => {
        controller.enqueue(`data: ${JSON.stringify({ id: id++ })}\n\n`);
      }, 1000);
      // close later if needed:
      // clearInterval(interval); controller.close();
    }
  });
  return new Response(stream, {
    headers: { 'content-type': 'text/event-stream' }
  });
}
```

## 11. URL and URLSearchParams

```js
// src/routes/search/+page.server.js
/** @type {import('./$types').PageServerLoad} */
export function load({ url }) {
  const query = url.searchParams.get('q') ?? '';
  const page = Number(url.searchParams.get('page') ?? '1');
  return { query, page };
}
```

```svelte
<!-- src/routes/search/+page.svelte -->
<script>
  /** @type {import('./$types').PageProps} */
  let { data } = $props();
</script>
<p>Searched for {data.query} (page {data.page})</p>
```

```js
// Build a URL programmatically
const url = new URL('/api/items', event.url);
url.searchParams.set('limit', '20');
fetch(url);
```

## 12. Web Crypto — generate UUIDs

```js
const uuid = crypto.randomUUID();
console.log(uuid);  // 'e3b0c442-...'
```

## 13. Web Crypto — hash a password

```js
const data = new TextEncoder().encode('hunter2');
const hash = await crypto.subtle.digest('SHA-256', data);
const hex = Array.from(new Uint8Array(hash))
  .map((b) => b.toString(16).padStart(2, '0'))
  .join('');
```

Note: prefer `argon2` / `bcrypt` for password hashing; SHA-256 alone is not sufficient.