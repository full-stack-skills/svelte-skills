# Hooks Reference

Three optional hook files:
- `src/hooks.server.js` — server only.
- `src/hooks.client.js` — client only.
- `src/hooks.js` — universal (runs on both).

## Server hooks (`hooks.server.js`)

### handle

```ts
export const handle: Handle = async ({ event, resolve }) => {
  // event: RequestEvent
  // resolve(event, opts?) => Promise<Response>
};
```

`resolve(event, { transformPageChunk?, filterSerializedResponseHeaders?, preload? })`:
- `transformPageChunk({ html, done }) => string | undefined`
- `filterSerializedResponseHeaders(name, value) => boolean`
- `preload({ type, path }) => boolean` — `type` is `'js' | 'css' | 'font' | 'asset'`

`resolve` never throws — always returns a `Response`.

### handleFetch

```ts
export const handleFetch: HandleFetch = async ({ event, request, fetch }) => {
  return fetch(request);
};
```

`fetch` here is the server fetch (with credentials handling). Same-origin requests get cookies forwarded; cross-origin subdomains also get cookies; sibling subdomains do NOT (manual `handleFetch` required).

### handleValidationError

```ts
export const handleValidationError: HandleValidationError = ({ event, issues }) => {
  return { message: 'Bad request' };
};
```

Customize the 400 response when a remote function arg fails Standard Schema validation.

## Shared hooks (`hooks.server.js` AND `hooks.client.js`)

### handleError

```ts
// Server
export const handleError: HandleServerError = async ({ error, event, status, message }) => {
  return { message: 'Whoops!', errorId: '...' };
};

// Client (note different type)
export const handleError: HandleClientError = async ({ error, event, status, message }) => {
  // event is NavigationEvent, not RequestEvent
  return { message: 'Whoops!' };
};
```

- Called for UNEXPECTED errors only (not `error(...)` thrown via `@sveltejs/kit`).
- Returned object becomes `page.error`.
- Must NEVER throw (wrap risky work in try/catch).

### init

```ts
// Server
export const init: ServerInit = async () => {
  await db.connect();
};

// Client
export const init: ClientInit = async () => {
  // Careful: async work delays hydration
};
```

Runs once at startup.

## Universal hooks (`hooks.js`)

### reroute

```ts
export const reroute: Reroute = ({ url, fetch }) => {
  // return pathname or undefined
};
```

- Async since 2.18.
- Must be pure/idempotent (cached per URL on client).
- Does NOT change the URL bar.
- Params derived from the returned pathname.

### transport

```ts
export const transport: Transport = {
  Vector: {
    encode: (value) => value instanceof Vector && [value.x, value.y],
    decode: ([x, y]) => new Vector(x, y)
  }
};
```

Custom types crossing the server/client boundary in `load` results and form action returns.

## sequence (helper)

```ts
import { sequence } from '@sveltejs/kit/hooks';

export const handle = sequence(first, second);
```

Behavior:
- `transformPageChunk` applied in REVERSE order, merged.
- `preload` applied in FORWARD order, first option wins.
- `filterSerializedResponseHeaders` same as `preload`.

## Init in client

Asynchronous work in `init` delays hydration. Use sparingly.

## Run during build

`building` from `$app/environment` is `true` during `vite build` and prerendering. Use to skip setup:

```js
import { building } from '$app/environment';
if (!building) {
  initialiseDatabase();
}
```

## $lib/server/ and init

`init` runs at server startup. Good place for:
- DB connection.
- Cache priming.
- Reading config files.

## Type imports

```ts
import type {
  Handle,
  HandleFetch,
  HandleValidationError,
  HandleServerError,
  HandleClientError,
  ServerInit,
  ClientInit,
  Reroute,
  Transport
} from '@sveltejs/kit';
```

## Common mistakes

- Throwing inside `handleError` → unhandled crash.
- Mutating response headers on a `Response.redirect()` → TypeError (immutable). Clone first or use a non-redirect Response.
- Treating `event.params` in `handleFetch` as authoritative → use for routing only, not auth.