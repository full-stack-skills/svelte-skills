# Errors Reference

## Error categories

| Type | Source | User sees | `page.error` | Renders |
|------|--------|-----------|--------------|---------|
| Expected | `error(status, payload)` | The payload (safe) | The payload | `+error.svelte` |
| Unexpected | any throw | `{ message: 'Internal Error' }` | Result of `handleError` | `+error.svelte` (or fallback) |
| Render | During component rendering | The `App.Error` prop | NOT updated | `+error.svelte` (opt-in flag) |

## Expected errors

```ts
import { error } from '@sveltejs/kit';

error(404, 'Not found');                                  // shorthand
error(404, { message: 'Not found', code: 'NOT_FOUND' });   // object form
```

- In SvelteKit 2.x: do NOT `throw` — just call.
- Sets the response status code.
- Renders the nearest `+error.svelte` with `page.error` populated.
- Use `redirect(status, location)` for navigation (not an error).

## Unexpected errors

```ts
throw new Error('Database lost');
```

- Default user message: `Internal Error` (status 500).
- Real message + stack go to console (dev) or server logs (prod).
- Routed through `handleError` hook.

## Rendering errors (opt-in)

Enable in `svelte.config.js`:

```js
kit: { experimental: { handleRenderingErrors: true } }
```

With this enabled, errors during SSR/CSR rendering invoke the nearest `+error.svelte`. The error is passed as a PROP (not via `page.error`):

```svelte
<!-- +error.svelte -->
<script>
  let { error, status } = $props();
</script>

<h1>{status}: {error.message}</h1>
```

Multiple `<svelte:boundary>` instances catch errors independently.

## +error.svelte

```svelte
<script>
  import { page } from '$app/state';
</script>

<h1>{page.status}</h1>
<p>{page.error.message}</p>
```

Nested error boundaries: closest `+error.svelte` above the failing load.

## Custom fallback page

```html
<!-- src/error.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>%sveltekit.error.message%</title>
  </head>
  <body>
    <h1>My error page</h1>
    <p>Status: %sveltekit.status%</p>
    <p>Message: %sveltekit.error.message%</p>
  </body>
</html>
```

Used when:
- Error in `handle` hook.
- Error in `+server.js`.
- Error in root `+layout.server.js` (root layout would normally contain `+error.svelte`).

JSON response (when `Accept` indicates API request) is automatic.

## App.Error type

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface Error {
      message: string;  // required
      // ...your fields
    }
  }
}
export {};
```

`page.error` and `+error.svelte`'s `error` prop are typed.

## handleError

```ts
// Server
export const handleError: HandleServerError = async ({ error, event, status, message }) => {
  // - error: the thrown error
  // - status: 500 for thrown errors
  // - message: 'Internal Error' (safe)
  // - event: RequestEvent
  return { message: 'Whoops!', errorId: crypto.randomUUID() };
};

// Client
export const handleError: HandleClientError = async ({ error, event, status, message }) => {
  // event is NavigationEvent
  return { message: 'Whoops!' };
};
```

Rules:
- Never throw inside `handleError`.
- Called for unexpected errors only.
- Returned object becomes `page.error`.

## handleValidationError

Customize 400 responses for failed remote function validation:

```ts
export const handleValidationError: HandleValidationError = ({ event, issues }) => {
  return { message: 'Nice try' };
};
```

## Redirects

```ts
import { redirect } from '@sveltejs/kit';

redirect(302, '/login');         // status, location
redirect(303, `/blog/${slug}`);  // post-create redirect
```

Allowed in:
- `+page.server.js` load/actions
- `+layout.server.js` load
- `+server.js`
- `query`, `form`, `prerender` remote functions

NOT allowed in:
- `command` remote functions (return `{ redirect: location }` instead)

## invalid (form-level validation)

```ts
import { invalid } from '@sveltejs/kit';

form(schema, async (data, issue) => {
  if (outOfStock) invalid(issue.qty('not enough'));
});
```

Throws (like `error`/`redirect`) but flags specific field issues.

## +error.svelte placement

- Place near the route where errors are expected.
- Layouts inherit child error boundaries via the route tree.
- Errors in `+layout.server.js` are caught by an `+error.svelte` ABOVE that layout, not next to it.

## Errors during SSR with no +error.svelte

If there's no `+error.svelte` and an expected error fires, SvelteKit falls back to a default error page. Add a custom `src/error.html` to control this.

## Common patterns

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface Locals { user: User | null; }
    interface PageData { user: User | null; }
    interface PageState { modal?: string }
    interface Error { message: string; code?: string; errorId?: string }
    interface Platform {}
  }
}
export {};
```