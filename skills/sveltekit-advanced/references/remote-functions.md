# Remote Functions Reference

Available since SvelteKit 2.27. Experimental — opt in via `kit.experimental.remoteFunctions` + `compilerOptions.experimental.async`.

## Setup

```js
// svelte.config.js
export default {
  kit: { experimental: { remoteFunctions: true } },
  compilerOptions: { experimental: { async: true } }
};
```

## File naming

Files named `*.remote.js` or `*.remote.ts` export remote functions. Cannot be placed inside `src/lib/server/`.

## query

```ts
import { query } from '$app/server';

const getThing = query(schema, async (arg) => Output);  // with validation
const getThing = query(async () => Output);              // no arg
const getThing = query('unchecked', async (arg) => Output);  // no validation
```

Returns a `RemoteQueryFunction`. Calling it returns a Promise-like with `.loading`, `.error`, `.current`, `.refresh()`.

```svelte
{#each await getThings() as item}
  ...
{/each}
```

- Args serialized via devalue → used as cache key.
- Object/Map/Set args are sorted (key order doesn't matter for cache).
- Server-side request-scoped cache dedupes.
- Client-side multiple identical calls share one instance.
- `.refresh()` re-fetches from server.

## query.batch

```ts
query.batch(schema, async (args[]) => (arg, idx) => Output);
```

- Collects calls within one macrotask.
- Solves n+1 query problems.
- Server callback receives array; returns a per-input resolver.

## query.live

```ts
query.live(async function* () {
  while (true) {
    yield value;
    await sleep(1000);
  }
});
```

- Returns an `AsyncIterable`.
- SSR returns only the FIRST yielded value, then closes.
- Client maintains connection while in active use.
- `.connected`, `.reconnect()`.
- No `.refresh()`.
- For-await over the instance directly is also supported.
- IMPORTANT: don't cache `no-store` responses in a service worker.

## form

```ts
form(schema, async (data, issue) => Output);
```

Returns a `RemoteForm` object spreadable on `<form {...form}>`.

Properties:
- `.method`, `.action` — works without JS.
- `.fields.<name>.as(type, value?)` — input attributes.
- `.fields.<name>.issues()` — validation issues.
- `.fields.<name>.value()` — current value.
- `.fields.<name>.set(value)` — programmatic update.
- `.fields.value()` — full form object.
- `.fields.set(obj)` — set all fields.
- `.fields.allIssues()` — combined form-level issues.
- `.validate({ includeUntouched? })` — revalidate.
- `.preflight(schema)` — client-side validation before submit.
- `.enhance(callback)` — custom submit handler.
- `.for(id)` — multiple instances.
- `.pending` — submission in progress.
- `.result` — last successful return value (ephemeral).
- `.element` — the `<form>` element (inside enhance).

Field naming:
- Underscore prefix (`_password`) → excluded from form repopulation.
- Boolean checkbox → use `v.optional(v.boolean(), false)` (not present when unchecked).
- File fields → `enctype="multipart/form-data"` required.

Programmatic validation:
```ts
import { invalid } from '@sveltejs/kit';
invalid(issue.qty('we don\'t have enough'));
```

## command

```ts
command(schema, async (arg) => Output);
```

- Cannot be called during render (only from event handlers).
- No `<form>` integration — JS-only.
- `redirect()` not allowed; return `{ redirect: location }` instead.

## prerender

```ts
prerender(schema, async (arg) => Output, { inputs?, dynamic? });
```

- Build-time execution; result cached.
- Excluded from server bundle unless `dynamic: true`.
- `inputs` returns array of arg values to prerender with.

## Validation

- Use any Standard Schema (Zod, Valibot, ArkType, etc.).
- On failure, SvelteKit returns 400 with generic message.
- Customize via `handleValidationError` hook.
- Opt out with the string `'unchecked'`.

## Redirects

- `query`, `form`, `prerender` — `redirect()` allowed.
- `command` — `redirect()` NOT allowed; return `{ redirect: location }`.

## Single-flight mutations

Server-driven refresh in mutation handler:
```ts
void getPosts().refresh();
getPost(id).set(updatedValue);
```

Client-requested:
```ts
for (const { query } of requested(getPosts, limit)) {
  void query.refresh();
}
// shorthand
await requested(getPosts, limit).refreshAll();
```

The `limit` is REQUIRED (DoS protection). Pass `Infinity` only with explicit intent.

## getRequestEvent

```ts
import { getRequestEvent, query } from '$app/server';

query(async () => {
  const { cookies, locals } = getRequestEvent();
  // ...
});
```

Notes:
- Cannot set headers (except cookies in `form`/`command`).
- `route`, `params`, `url` reflect the CALLER'S page, not the endpoint.
- Never authorize based on `params`/`url` here.

## Imports

```ts
import {
  command,        // command()
  form,           // form()
  getRequestEvent,
  prerender,      // prerender()
  query,          // query() + query.batch + query.live
  read,           // read imported assets
  requested       // requested for client-requested refreshes
} from '$app/server';
```