# Routing Reference

SvelteKit uses a filesystem router. The `src/routes` directory defines the URL tree. Route files are identified by their `+` prefix.

## Three rules to memorize

1. All route files can run on the server.
2. All route files run on the client EXCEPT `+server.js`.
3. `+layout` and `+error` files apply to the directory they live in AND all subdirectories.

## Route file types

| File | Runs on | Purpose |
|---|---|---|
| `+page.svelte` | server + client | Page UI |
| `+page.js` | server + client | Universal `load` and page options |
| `+page.server.js` | server only | Server `load`, form `actions`, page options |
| `+layout.svelte` | server + client | Wraps pages and nested layouts |
| `+layout.js` | server + client | Universal `load` for layout |
| `+layout.server.js` | server only | Server `load` for layout |
| `+server.js` | server only | API endpoint (HTTP verb handlers) |
| `+error.svelte` | server + client | Per-route error boundary |

Any other file in a route folder is ignored by the router — use it for colocated components or utilities.

## Folder-name conventions

| Folder name | Meaning |
|---|---|
| `about/` | Static segment `/about` |
| `[slug]/` | Required param |
| `[[slug]]/` | Optional param |
| `[...rest]/` | Rest param (greedy) |
| `[slug=int]/` | Param with matcher named `int` from `src/params/` |
| `(group)/` | Grouping (not part of URL) |
| `[slug=integer]/` | Same as above, with `integer` matcher |

## +page triad

### +page.svelte

```svelte
<script>
  /** @type {import('./$types').PageProps} */
  let { data, params, form } = $props();
</script>
```

- `data` — value returned from `load`.
- `params` — typed route params (since 2.24).
- `form` — `ActionData` (form submission result), for `+page.server.js` only.

### +page.js (universal)

```js
/** @type {import('./$types').PageLoad} */
export async function load({ params, fetch, url, route, depends }) {
  // runs on server during SSR, browser during CSR
  return { foo: 'bar' };
}

// Page options (per-route overrides)
export const prerender = true;
export const ssr = true;
export const csr = true;
```

### +page.server.js (server-only)

```js
/** @type {import('./$types').PageServerLoad} */
export async function load({ params, locals, cookies }) {
  return { foo: 'bar' };
}

/** @type {import('./$types').Actions} */
export const actions = {
  default: async ({ request }) => { /* ... */ },
  named: async ({ request }) => { /* ... */ }
};
```

The returned value is serialized with [devalue](https://github.com/Rich-Harris/devalue) for CSR; functions, class instances, etc. need explicit handling.

## +layout triad

Same structure as `+page`, with `LayoutProps`, `LayoutLoad`, `LayoutServerLoad`.

A `+layout.svelte` MUST render `{@render children()}` somewhere.

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  let { children, data } = $props();
</script>
<nav>...</nav>
{@render children()}
```

Layout data is inherited by all child pages. Page-level `data` is merged on top.

## +server.js

```js
/** @type {import('./$types').RequestHandler} */
export function GET({ request, url, params, cookies, fetch }) {
  return new Response('hello');
}
```

Exported handlers: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, `HEAD`, and `fallback` (matches anything else).

`+server.js` files do NOT inherit `+layout.svelte`. They emit raw responses.

## +error.svelte

```svelte
<!-- src/routes/blog/[slug]/+error.svelte -->
<script>
  import { page } from '$app/state';
</script>
<h1>{page.status}: {page.error.message}</h1>
```

Boundary lookup walks UP the tree:

1. Same folder as the error.
2. Parent folder, recursively.
3. Root `src/routes/+error.svelte`.
4. Static fallback `src/error.html`.

`+error.svelte` is NOT used for errors in `+server.js` handlers or in the `handle` hook — those use `src/error.html`.

## Content negotiation

When `+page.svelte` and `+server.js` share a folder:

- `PUT` / `PATCH` / `DELETE` / `OPTIONS` -> `+server.js` (pages can't accept these).
- `GET` / `POST` / `HEAD`:
  - If `Accept` header prioritizes `text/html` -> page (HTML).
  - Otherwise -> `+server.js` (data).
- `GET` responses include `Vary: Accept`.

## $types

SvelteKit generates `./$types.d.ts` per route folder. Types:

| Type | Used in | What it types |
|---|---|---|
| `PageLoad` | `+page.js` | The `load` function |
| `PageServerLoad` | `+page.server.js` | The `load` function |
| `LayoutLoad` | `+layout.js` | The `load` function |
| `LayoutServerLoad` | `+layout.server.js` | The `load` function |
| `Actions` | `+page.server.js` | The `actions` export |
| `RequestHandler` | `+server.js` | Each HTTP verb handler |
| `PageProps` | `+page.svelte` | The `$props()` value (`{ data, params, form }`) |
| `LayoutProps` | `+layout.svelte` | The `$props()` value (`{ data, children }`) |
| `PageData`, `LayoutData`, `ActionData`, `ErrorData` | narrower typing | individual prop types |

`PageProps` / `LayoutProps` were added in 2.16. With VS Code + Svelte extension, types are inferred automatically.

## Page options

Exported from `+page.js` or `+page.server.js`:

```js
export const prerender = true;   // 'auto' | true | false
export const ssr = true;         // boolean
export const csr = true;         // boolean
```

Inherited down the tree when set in `+layout.js`. Override per-page.

## Navigation

- `<a href="/x">` — client-side navigation (default).
- `<a href="/x" data-sveltekit-reload>` — full page reload.
- `<a href="/x" data-sveltekit-preload-data="hover">` — preload on hover.
- `<a href="/x" data-sveltekit-noscroll>` — don't reset scroll position.

Programmatic:

```js
import { goto, invalidate, invalidateAll, beforeNavigate, afterNavigate } from '$app/navigation';
goto('/x');
invalidate('/api/me');
invalidateAll();
```

## Further reading

- [Tutorial: Routing](/tutorial/kit/pages)
- [Tutorial: API routes](/tutorial/kit/get-handlers)
- [Docs: Advanced routing](advanced-routing) — rest params, optional params, matchers, route groups
- [Docs: Form actions](form-actions)