# Web Standards Reference

SvelteKit "uses the platform". Every API listed here is a browser-native Web API that SvelteKit makes available in `load` functions, hooks, `+server.js` handlers, and the browser.

## Fetch APIs

### `fetch(input, init?)`

Standard [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/fetch).

In SvelteKit, `fetch` is enhanced inside `load` functions, `+server.js` handlers, server hooks, and remote functions:

- **Relative URLs work** — `fetch('/api/items')` resolves against the current page URL.
- **Credentials auto-forward** — cookies are forwarded for same-origin requests; for cross-origin, you must pass `cookie`/`authorization` headers yourself.

Outside `load`/server handlers, `fetch` is the global browser/Node fetch (no credentials forwarding).

```js
// +page.js  -- fetch is enhanced
export async function load({ fetch }) {
  const res = await fetch('/api/me');         // relative
  const data = await fetch('https://api.example.com/data');   // absolute
}

// +server.js handler  -- fetch is also enhanced here
export async function POST({ request, fetch }) {
  const res = await fetch('https://internal-service/data', {
    headers: { authorization: `Bearer ${env.API_KEY}` }
  });
}
```

### `Request`

[Request](https://developer.mozilla.org/en-US/docs/Web/API/Request) — represents the incoming HTTP request.

Available as `event.request` in `+server.js` handlers and hooks.

Methods:

- `request.json()` — parse JSON body
- `request.text()` — raw text
- `request.formData()` — parse `multipart/form-data` or `application/x-www-form-urlencoded`
- `request.arrayBuffer()` — bytes
- `request.blob()` — Blob
- `request.headers` — `Headers` object
- `request.method`, `request.url`

### `Response`

[Response](https://developer.mozilla.org/en-US/docs/Web/API/Response) — the value every `+server.js` handler must return.

```js
return new Response(body, { status, headers });
return new Response(stream, { headers: { 'content-type': 'text/event-stream' } });
```

Helpers from `@sveltejs/kit`:

- `json(data, init?)` — `Response` with `content-type: application/json`.
- `text(body, init?)` — `Response` with `content-type: text/plain`.
- `redirect(status, location)` — throws a redirect (use `throw` or call; both work).
- `error(status, message)` — throws an error (use `throw` or call; both work).
- `devalue.unparse(data)` — for serializing complex payloads (used internally).

### `Headers`

[Headers](https://developer.mozilla.org/en-US/docs/Web/API/Headers) — read `request.headers`, set `response.headers`.

```js
const ua = request.headers.get('user-agent');
return json(data, { headers: { 'x-foo': 'bar' } });
```

In SvelteKit, the `Set-Cookie` header should be set via `event.cookies.set(...)`, not via raw headers.

## FormData

[FormData](https://developer.mozilla.org/en-US/docs/Web/API/FormData) — used for HTML form submissions and `multipart/form-data`.

```js
const data = await request.formData();
data.get('name');           // string | File | null
data.has('name');
data.getAll('tags');
[...data.entries()];        // [['name', 'Ada'], ...]
```

For multiple values per key, use `data.getAll('tag')`.

## Stream APIs

For endpoints that return large or chunked data:

- [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) — produces a stream of chunks.
- [`WritableStream`](https://developer.mozilla.org/en-US/docs/Web/API/WritableStream) — consumes chunks.
- [`TransformStream`](https://developer.mozilla.org/en-US/docs/Web/API/TransformStream) — transforms streams.

Used in `+server.js` for streaming responses and server-sent events:

```js
const stream = new ReadableStream({
  start(controller) {
    controller.enqueue(new TextEncoder().encode('chunk'));
    controller.close();
  }
});
return new Response(stream, { headers: { 'content-type': 'text/plain' } });
```

Note: some serverless platforms (AWS Lambda) buffer responses — streaming won't work there without specific configuration.

## URL APIs

[URL](https://developer.mozilla.org/en-US/docs/Web/API/URL) — represents a parsed URL.

In SvelteKit:

- `event.url` — `URL` in hooks and `+server.js`.
- `page.url` — `URL` on the page (from `$app/state` in Svelte 5 / `$app/stores` in Svelte 4).
- `from.url`, `to.url` — in `beforeNavigate` / `afterNavigate`.

```js
const url = new URL('/api/items', event.url);
url.searchParams.set('limit', '20');
fetch(url);
```

### URLSearchParams

[URLSearchParams](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams) — query-string helper.

```js
const q = url.searchParams.get('q');          // string | null
url.searchParams.set('page', '2');
url.searchParams.has('debug');
url.searchParams.toString();                  // 'page=2&sort=desc'
```

## Web Crypto

[Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API) — accessible via the `crypto` global.

Common uses:

```js
// Random UUID
const id = crypto.randomUUID();

// Random bytes
const bytes = crypto.getRandomValues(new Uint8Array(32));

// Hash
const hash = await crypto.subtle.digest('SHA-256', new TextEncoder().encode('text'));

// HMAC
const key = await crypto.subtle.importKey('raw', secretBytes, { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']);
const sig = await crypto.subtle.sign('HMAC', key, data);
```

SvelteKit uses `crypto` internally for CSP nonces.

## RequestEvent (SvelteKit-specific)

The `RequestEvent` argument to handlers in `+server.js`, hooks, and `load` exposes:

| Property | Type | Description |
|---|---|---|
| `request` | `Request` | Incoming request |
| `url` | `URL` | Parsed URL |
| `params` | object | Route params |
| `route` | `{ id }` | Route identifier |
| `cookies` | `Cookies` | Read/write cookies |
| `fetch` | enhanced `fetch` | SvelteKit fetch |
| `getClientAddress` | `() => string` | IP (when supported by platform) |
| `locals` | object | Per-request locals (set in hooks) |
| `platform` | object | Platform-specific data (Cloudflare, Vercel) |
| `setHeaders` | `(headers) => void` | Set headers on the cached response |

## Polyfills

Modern Node (18+) ships with most of these APIs natively. Older Node versions and certain adapters get polyfills automatically. Don't add your own polyfills — duplicate globals cause subtle bugs.