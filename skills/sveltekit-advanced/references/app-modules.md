# $app/* Modules Reference

## $app/forms (legacy form actions)

```ts
import { applyAction, deserialize, enhance } from '$app/forms';
```

- `enhance(formElement, submitFn?)` — progressively enhances a `<form method="POST" action="?/action">`.
  - Returns `{ destroy() }`.
  - `submitFn({ formData, action, cancel, controller, submitter })` — return a function to handle the result.
  - Default behavior: updates `page.form`, `page.status`, invalidates all data, redirects on redirect result.
  - `update({ reset?, invalidateAll? })` — call from your callback for default behavior.

- `applyAction(result)` — update `page.form` and `page.status`, redirect on error.
- `deserialize(text)` — parse a form action response string into `ActionResult`.

## $app/navigation

```ts
import {
  afterNavigate, beforeNavigate, disableScrollHandling, goto,
  invalidate, invalidateAll, onNavigate, preloadCode, preloadData,
  pushState, refreshAll, replaceState
} from '$app/navigation';
```

### goto

```ts
goto(url: string | URL, {
  replaceState?: boolean,
  noScroll?: boolean,
  keepFocus?: boolean,
  invalidateAll?: boolean,
  invalidate?: (string | URL | (url => boolean))[],
  state?: App.PageState
}): Promise<void>
```

### invalidate / invalidateAll / refreshAll

- `invalidate(resource)` — re-run loads that depend on this URL or predicate.
- `invalidateAll()` — re-run all active `load` functions.
- `refreshAll({ includeLoadFunctions?: boolean })` — re-run all active queries AND optionally `load`.

Custom predicate: `invalidate((url) => url.pathname === '/api/x')`.
Custom identifier: `invalidate('custom:thing')` — must match `[a-z]+:` pattern.

### preloadCode / preloadData

- `preloadCode(pathname)` — import route code.
- `preloadData(href)` — import code + run `load`. Returns `{ type: 'loaded', status, data } | { type: 'redirect', location }`.

### pushState / replaceState

```ts
pushState(url: string | URL, state: App.PageState): void
replaceState(url: string | URL, state: App.PageState): void
```

For shallow routing. URL `''` keeps current URL.

### beforeNavigate / afterNavigate / onNavigate

Must be called during component initialization (top-level `<script>` or in `onMount`-like).

- `beforeNavigate(({ cancel, type, from, to, willUnload }) => void)`.
- `afterNavigate(({ from, to, type }) => void)`.
- `onNavigate(...)` — can return a Promise (e.g., `document.startViewTransition`).

`type` is `'enter' | 'form' | 'link' | 'goto' | 'popstate' | 'leave'`.

### disableScrollHandling

Call from `afterNavigate` or `onMount` to disable SvelteKit's scroll-to-top.

## $app/state (Svelte 5, since 2.12)

```ts
import { navigating, page, updated } from '$app/state';
```

- `page` — reactive read-only object.
  - `page.url: URL`
  - `page.params: Record<string, string>`
  - `page.route: { id: RouteId | null }`
  - `page.status: number`
  - `page.error: App.Error | null`
  - `page.data: PageData` — merged data from all loads
  - `page.form: ActionData | null` — last form result
  - `page.state: App.PageState` — set via pushState/replaceState/goto
- `navigating` — `Navigation | null`. During SSR/prerender, always null.
- `updated` — `{ current: boolean, check(): Promise<boolean> }`. Polls for new app version if `version.pollInterval > 0`.

Use in templates or `$derived(...)`. On the server, read only during rendering.

## $app/stores (deprecated since 2.12)

Same exports as `$app/state` but as Svelte stores. Use `$app/state` for new code with Svelte 5.

## $app/paths

```ts
import { asset, match, resolve } from '$app/paths';
```

- `asset(file: Asset): string` — resolve static file URL (handles base path).
- `resolve(pathname | (routeId, params))` — resolve path with base path applied.
- `match(url | pathname): Promise<{ id: RouteId, params } | null>` — match a path to a route ID.

Deprecated: `assets`, `base`, `resolveRoute`. Use `asset()` and `resolve()`.

## $app/server

```ts
import {
  command, form, getRequestEvent, prerender, query, read, requested
} from '$app/server';
```

- `query(schema, fn)` / `query(fn)` / `query('unchecked', fn)`.
- `query.batch(...)`, `query.live(...)`.
- `form(schema, fn)` / `form('unchecked', fn)` / `form(fn)`.
- `command(...)` — like form but no element.
- `prerender(schema, fn, opts?)` — opts: `{ inputs, dynamic }`.
- `read(asset)` — read imported asset from filesystem. Returns `Response`.
- `getRequestEvent()` — current `RequestEvent`.
- `requested(queryFn, limit)` — for client-requested refreshes inside mutations.

## $app/environment

```ts
import { browser, building, dev, version } from '$app/environment';
```

- `browser: boolean` — true on client.
- `building: boolean` — true during vite build and prerender.
- `dev: boolean` — dev server running.
- `version: string` — `config.kit.version.name`.

## $app/env (explicit env vars only)

Same as `$app/environment`. Plus `$app/env/private` and `$app/env/public` for explicit env vars.

## $app/types

```ts
import type { RouteId, Pathname, RouteParams, LayoutParams, Asset } from '$app/types';
```

- `RouteId` — union of all route IDs (e.g., `'/' | '/blog/[slug]'`).
- `Pathname` — union of all valid pathnames.
- `RouteParams<T extends RouteId>` — `{ slug: string } | Record<string, never>`.
- `LayoutParams<T>` — like RouteParams but optional for child routes.
- `Asset` — union of static filenames + wildcard string.

## $lib alias

```js
import Component from '$lib/Component.svelte';
// resolves to src/lib/Component.svelte
```

Configurable via `kit.files.lib` (default `'src/lib'`).

## $service-worker

```ts
import { base, build, files, prerendered, version } from '$service-worker';
```

Only available inside service workers. See `service-worker.md` reference.