# Server-Only Modules Reference

## Built-in server-only modules

- `$env/static/private`
- `$env/dynamic/private`
- `$app/server` (includes `read`, `getRequestEvent`, remote function helpers)

Cannot be imported from browser code.

## Making your own modules server-only

Two ways:

1. **Filename suffix**: `secrets.server.js`
2. **Location**: `$lib/server/secrets.js`

Both prevent browser imports.

## How detection works

SvelteKit analyzes the import graph. If a browser-bound file transitively imports a server-only module, the build errors:

```
Cannot import $lib/server/secrets.ts into code that runs in the browser,
as this could leak sensitive information.

 src/routes/+page.svelte imports
  src/routes/utils.js imports
   $lib/server/secrets.ts

If you're only using the import as a type, change it to `import type`.
```

Even if you only import non-secret values, the chain is still considered unsafe. Use `import type` for type-only imports.

## Dynamic imports also checked

```js
const mod = await import(`./${foo}.js`);  // still checked
```

## Test mode

Detection disabled when `process.env.TEST === 'true'`. Vitest and similar tools won't trigger false positives.

## Why use $lib/server?

- Clear physical separation.
- Anything inside is implicitly server-only (no `.server` suffix needed).
- Convention; preferred for new projects.

## Why use .server suffix?

- Co-locate server and client code with the same base name.
- Useful when a module has both server and public exports in different files.

## Common mistake

```js
// utils.js (PUBLIC)
export { secret } from '$lib/server/secrets.js';  // BAD
export const add = (a, b) => a + b;
```

Even if `+page.svelte` only imports `add`, the entire `utils.js` is suspect and the build fails.

Fix: split into two files, or use `import type`.

## Type-only imports

```ts
import type { User } from '$lib/server/database';
```

Allowed in client code — types are erased at compile time.

## $app/server

```ts
import { read, getRequestEvent, query, form, command, prerender, requested } from '$app/server';
```

`read(asset)` — read an imported asset from the filesystem (server-only).

`getRequestEvent()` — current `RequestEvent` for cookies/locals access.

## Private env vars

```js
// Server only
import { env } from '$env/dynamic/private';
import { API_KEY } from '$env/static/private';
```

Public vars (`PUBLIC_*`) can also be imported here.

## In route files

- `+page.server.js`, `+layout.server.js` — server-only by file name. Can import anything.
- `+page.js`, `+layout.js` — universal; CANNOT import server-only modules.
- `+server.js` — server-only by file name.

## In hooks

- `hooks.server.js` — server-only.
- `hooks.client.js` — client-only.
- `hooks.js` — universal; CANNOT import server-only.

## Security reminder

Server-only modules prevent ACCIDENTAL leaks via the import graph. They don't replace:
- Auth/authz (still need checks).
- HTTPS.
- Input validation.
- Rate limiting.

## Recommended structure

```
src/lib/
  server/             <- server-only
    database.js
    auth.js
    secrets.js
  components/         <- shared
  utils.js            <- shared
```