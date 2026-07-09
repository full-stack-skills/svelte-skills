# Errors Examples

Expected vs unexpected errors in SvelteKit, with custom error pages and `App.Error` typing.

## 1. Expected error from load

```js
// src/routes/blog/[slug]/+page.server.js
import { error } from '@sveltejs/kit';
import * as db from '$lib/server/database';

export async function load({ params }) {
  const post = await db.getPost(params.slug);
  if (!post) error(404, { message: 'Not found' });
  return { post };
}
```

In SvelteKit 2.x you do NOT throw `error(...)` — calling it is enough.

## 2. +error.svelte

```svelte
<!-- src/routes/+error.svelte -->
<script>
  import { page } from '$app/state';
</script>

<h1>{page.error.message}</h1>
{#if page.error.code}
  <p>Code: {page.error.code}</p>
{/if}
```

`page.error` is whatever you passed to `error(status, payload)`.

## 3. Error with extra fields

```js
error(404, {
  message: 'Not found',
  code: 'NOT_FOUND',
  trackingId: crypto.randomUUID()
});
```

Type the fields via `App.Error` (see #8).

## 4. Error with shorthand string

```js
import { error } from '@sveltejs/kit';
error(404, 'Not found');  // same as error(404, { message: 'Not found' })
```

## 5. Unexpected error (default behavior)

```js
export async function load() {
  throw new Error('Database connection lost');
  // User sees: { message: 'Internal Error' }
  // Real message goes to console / server logs
}
```

Unexpected errors are NOT shown to the user.

## 6. handleError (server) — Sentry + custom shape

```js
// src/hooks.server.js
import * as Sentry from '@sentry/sveltekit';

Sentry.init({/* ... */});

export async function handleError({ error, event, status, message }) {
  const errorId = crypto.randomUUID();
  Sentry.captureException(error, { extra: { event, errorId, status } });
  return { message: 'Whoops!', errorId };
}
```

`handleError` is called for unexpected errors ONLY.

## 7. handleError (client)

```js
// src/hooks.client.js
import * as Sentry from '@sentry/sveltekit';

Sentry.init({/* ... */});

export async function handleError({ error, event, status, message }) {
  const errorId = crypto.randomUUID();
  Sentry.captureException(error, { extra: { event, errorId, status } });
  return { message: 'Whoops!', errorId };
}
```

Note: client uses `HandleClientError`; `event` is `NavigationEvent`.

## 8. App.Error interface

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface Error {
      message: string;  // required
      code: string;
      errorId: string;
    }
  }
}
export {};
```

Now `page.error` and `error.status` props are typed.

## 9. Rendering error boundary (opt-in)

```js
// svelte.config.js
export default {
  kit: { experimental: { handleRenderingErrors: true } }
};
```

With this enabled, errors during rendering show `+error.svelte`. The error is passed as a PROP, not via `page.error`:

```svelte
<!-- +error.svelte -->
<script>
  let { error } = $props();
</script>
<h1>{error.message}</h1>
```

`page.error` is NOT updated for render errors (page already started rendering).

## 10. svelte:boundary failed snippet

```svelte
<svelte:boundary>
  {#snippet failed(error: App.Error)}
    {error.message}
  {/snippet}
  <!-- ... -->
</svelte:boundary>
```

## 11. Custom error.html

```html
<!-- src/error.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>%sveltekit.error.message%</title>
  </head>
  <body>
    <h1>My custom error page</h1>
    <p>Status: %sveltekit.status%</p>
    <p>Message: %sveltekit.error.message%</p>
  </body>
</html>
```

Used for errors in `handle`, `+server.js`, or root `+layout.server.js` (since the root layout would normally contain `+error.svelte`).

## 12. Redirect from load (not error)

```js
import { redirect } from '@sveltejs/kit';

export async function load({ locals }) {
  if (!locals.user) redirect(302, '/login');
  return { user: locals.user };
}
```

## 13. Redirect inside a form/command

```js
import { redirect } from '@sveltejs/kit';

export const createPost = form(schema, async (data) => {
  await db.insert(data);
  redirect(303, `/blog/${data.slug}`);
});
```

For `command`, return `{ redirect: location }` instead.

## 14. JSON vs HTML fallback

By default SvelteKit returns JSON for non-browser Accept headers. Custom `src/error.html` covers the HTML fallback case.

## 15. Try/catch in +server.js

```js
import { json, error } from '@sveltejs/kit';

export async function GET({ url }) {
  try {
    const data = await db.fetch(url.searchParams.get('id'));
    if (!data) error(404, 'Not found');
    return json(data);
  } catch (e) {
    throw e;  // unexpected — handled by handleError
  }
}
```

## 16. Custom 500 error page

```svelte
<!-- src/routes/+error.svelte -->
<script>
  import { page } from '$app/state';
</script>

{#if page.status === 404}
  <h1>Page not found</h1>
{:else if page.status === 500}
  <h1>Server error</h1>
  <p>Something went wrong on our end.</p>
{:else}
  <h1>Error {page.status}</h1>
{/if}
```

## 17. handleError must never throw

```js
// BAD
export async function handleError({ error }) {
  await logger.send(error);  // if logger throws, app crashes
}

// GOOD
export async function handleError({ error }) {
  try {
    await logger.send(error);
  } catch (e) {
    console.error('logger failed', e);
  }
  return { message: 'Whoops!' };
}
```

## 18. Validation error from remote function

```js
export function handleValidationError({ issues }) {
  return { message: 'Invalid request' };
}
```

Only called when remote function arg fails Standard Schema validation.