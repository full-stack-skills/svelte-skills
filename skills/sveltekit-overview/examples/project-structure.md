# Project Structure Examples

The standard SvelteKit layout produced by `npx sv create`. Examples show what goes where.

## 1. Minimal layout

```tree
my-app/
├ src/
│  ├ routes/
│  │  └ +page.svelte
│  └ app.html
├ static/
│  └ favicon.png
├ package.json
├ svelte.config.js
├ tsconfig.json
└ vite.config.js
```

## 2. Adding a library folder

```tree
src/
├ lib/
│  ├ server/
│  │  └ db.ts        # server-only (uses $env/private, DB clients)
│  ├ components/
│  │  └ Button.svelte
│  └ utils/
│     └ format.ts
├ routes/
│  └ +page.svelte
└ app.html
```

```svelte
<!-- src/lib/components/Button.svelte -->
<script>
  let { children, ...rest } = $props();
</script>
<button {...rest}>{@render children()}</button>
```

Use anywhere via `import Button from '$lib/components/Button.svelte'`.

## 3. Server-only lib guard

```tree
src/lib/server/db.ts    # <- this folder is server-only
```

```ts
// src/lib/server/db.ts
import { DATABASE_URL } from '$env/static/private';
import { createClient } from '@supabase/supabase-js';
export const db = createClient(DATABASE_URL);
```

Importing `$lib/server/db` from a client file causes a build error — that's the point.

## 4. Param matchers

```tree
src/params/
└ integer.ts
```

```ts
// src/params/integer.ts
export function match(value) {
  return /^\d+$/.test(value);
}
```

```tree
src/routes/items/
└ [id=int]/+page.svelte
```

## 5. Static assets

```tree
static/
├ favicon.ico
├ robots.txt
└ images/
   └ logo.svg
```

Referred to as `/favicon.ico`, `/robots.txt`, `/images/logo.svg`. Prefer `import logo from './logo.svg'` over `static/` so Vite can hash them.

## 6. Hooks

```tree
src/
├ hooks.client.js
├ hooks.server.js
└ service-worker.js
```

```js
// src/hooks.server.js
/** @type {import('@sveltejs/kit').Handle} */
export async function handle({ event, resolve }) {
  // e.g. set locals, log, auth
  return resolve(event);
}
```

## 7. Custom error page

```html
<!-- src/error.html -->
<!doctype html>
<html>
  <head><title>%sveltekit.error.message%</title></head>
  <body>
    <h1>%sveltekit.status%</h1>
    <p>%sveltekit.error.message%</p>
  </body>
</html>
```

## 8. Custom app.html

```html
<!-- src/app.html -->
<!doctype html>
<html lang="%sveltekit.env.LANG%">
  <head>
    <meta charset="utf-8" />
    <link rel="icon" href="%sveltekit.assets%/favicon.png" />
    %sveltekit.head%
  </head>
  <body data-sveltekit-preload-data="hover">
    <div style="display: contents">%sveltekit.body%</div>
  </body>
</html>
```

`%sveltekit.body%` MUST be inside a wrapper element (not directly in `<body>`) to avoid hydration bugs.

## 9. Playwright tests folder

```tree
tests/
├ test.ts          # global setup
└ homepage.spec.ts # spec file
```

```ts
// tests/homepage.spec.ts
import { test, expect } from '@playwright/test';

test('homepage has h1', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('h1')).toBeVisible();
});
```

## 10. Inline route components

```tree
src/routes/blog/
├ +page.svelte
├ +page.server.js
└ CommentBox.svelte     # co-located; not a route
```

Files without the `+` prefix in a route folder are ignored by the router. Use them for components that only one route needs.