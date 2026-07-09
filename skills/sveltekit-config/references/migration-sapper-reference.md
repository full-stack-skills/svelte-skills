# Sapper → SvelteKit migration reference

Complete guide for migrating a Sapper app to SvelteKit. Sapper 0.29.3+ allows incremental migration.

## package.json changes

### Add `"type": "module"`

```diff
{
+  "type": "module",
   "name": "my-app",
   ...
}
```

Can be done separately from the rest of the migration if you're on Sapper 0.29.3+.

### Remove dependencies

```diff
{
-  "dependencies": {
-    "polka": "^1.0.0",
-    "sirv": "^1.0.0",
-    "compression": "^1.7.0"
-  }
}
```

(Or `express` instead of `polka` + `sirv`.)

### Replace devDependencies

```diff
{
-  "devDependencies": {
-    "sapper": "^0.29.0",
-    "webpack": "^5.0.0",
-    "rollup": "^2.0.0"
-  },
+  "devDependencies": {
+    "@sveltejs/kit": "^2.0.0",
+    "@sveltejs/adapter-node": "^5.0.0",
+    "vite": "^5.0.0"
+  }
}
```

### Update scripts

```diff
{
-  "scripts": {
-    "dev": "sapper dev",
-    "build": "sapper build",
-    "export": "sapper export",
-    "start": "node __sapper__/build"
-  }
+  "scripts": {
+    "dev": "vite dev",
+    "build": "vite build",
+    "preview": "vite preview",
+    "start": "node build"
+  }
}
```

## Project file changes

### `webpack.config.js` / `rollup.config.js` → `svelte.config.js`

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-node';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

/** @type {import('@sveltejs/kit').Config} */
export default {
  preprocess: vitePreprocess(),
  kit: { adapter: adapter() }
};
```

### `src/client.js`

No equivalent. Move `onMount` logic into `+layout.svelte`:

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import { onMount } from 'svelte';
  onMount(() => {
    // client-only setup
  });
</script>
```

### `src/server.js`

Equivalent: [custom Node server](adapter-node#custom-server). Otherwise no direct equivalent (SvelteKit is platform-agnostic).

### `src/service-worker.js`

Imports from `@sapper/service-worker` change to `$service-worker`:

| Sapper | SvelteKit |
|--------|-----------|
| `files` | `files` |
| `routes` | removed |
| `shell` | `build` |
| `timestamp` | `version` |

### `src/template.html` → `src/app.html`

```diff
-<html>
+<!DOCTYPE html>
+<html lang="en">
   <head>
-    <meta charset="utf-8" />
-    <meta name="viewport" ... />
-    <title>%sapper.title%</title>
-
-    %sapper.base%
-    %sapper.styles%
-    %sapper.head%
+    <meta charset="utf-8" />
+    <meta name="viewport" ... />
+    <title>My App</title>
+    %sveltekit.head%
   </head>
   <body>
-    <div id="sapper">%sapper.html%</div>
-    %sapper.scripts%
+    <div style="display: contents">%sveltekit.body%</div>
   </body>
 </html>
```

### `src/node_modules` → `src/lib`

Move `src/node_modules/*` to `src/lib/`. Update imports.

## Routes & layouts

### File renaming

```tree
routes/about/index.svelte    →  routes/about/+page.svelte
routes/about.svelte          →  routes/about/+page.svelte
routes/_layout.svelte        →  routes/+layout.svelte
routes/_error.svelte         →  routes/+error.svelte
```

### Imports

```diff
-import { goto, prefetch, prefetchRoutes } from '@sapper/app';
+import { goto, preloadData, preloadCode } from '$app/navigation';

-import { stores } from '@sapper/app';
-const { preloading, page, session } = stores();
+import { page, navigating } from '$app/state';  // or $app/stores (legacy)
```

### `preload` → `load`

```diff
-// Sapper: preload({ page, session }) { ... }
-// this.fetch, this.error, this.redirect available

+// SvelteKit: export function load(event) { ... }
+// event.fetch, error(), redirect() imported from @sveltejs/kit
```

```diff
-import { error } from '@sapper/app';
+import { error } from '@sveltejs/kit';

-throw new Error({ code: 404 });
+error(404, { message: 'Not found' });
```

### Routing

Regex routes → [matchers](advanced-routing#matching).

`src/params/integer.js`:

```js
export const match = (param) => /^\d+$/.test(param);
```

```tree
src/routes/items/[id=integer]/+page.svelte
```

### `<a>` attributes

```diff
-<a sapper:prefetch href="/about">About</a>
+<a data-sveltekit-preload-data href="/about">About</a>

-<a sapper:noscroll href="/chat">Chat</a>
+<a data-sveltekit-noscroll href="/chat">Chat</a>
```

### URL resolution

Sapper resolved relative URLs against the base URL. SvelteKit resolves against the current page. Use root-relative URLs (`/about`) for predictability.

## Endpoints

```diff
-// Sapper
-export function get(req, res, next) {
-  res.writeHead(200, { 'Content-Type': 'application/json' });
-  res.end(JSON.stringify({ hello: 'world' }));
-}

+// SvelteKit
+/** @type {import('./$types').RequestHandler} */
+export function GET() {
+  return json({ hello: 'world' });
+}
```

`fetch` is now global — no need for `node-fetch` / `cross-fetch`.

## HTML minification

Add via hook:

```js
// src/hooks.server.js
import { minify } from 'html-minifier';
import { building } from '$app/environment';

const opts = {
  collapseBooleanAttributes: true,
  collapseWhitespace: true,
  conservativeCollapse: true,
  decodeEntities: true,
  html5: true,
  minifyCSS: true,
  minifyJS: false,  // hydration needs comments
  removeAttributeQuotes: true,
  removeOptionalTags: true,
  removeRedundantAttributes: true,
  removeScriptTypeAttributes: true,
  removeStyleLinkTypeAttributes: true,
  sortAttributes: true,
  sortClassName: true
};

export async function handle({ event, resolve }) {
  let page = '';
  return resolve(event, {
    transformPageChunk: ({ html, done }) => {
      page += html;
      if (done) return building ? minify(page, opts) : page;
    }
  });
}
```

## Migration strategy

### Incremental (recommended)

1. Update package.json scripts to use `vite dev` and `vite build` with `adapter-node`
2. Move routes progressively:
   - Rename `_layout.svelte` → `+layout.svelte`, etc.
   - Convert `preload` → `load`
   - Test
3. Once all routes converted, remove Sapper entirely

### Big bang

If the app is small enough, do all renames + load conversions at once. Test thoroughly.

## See also

- [examples/migration-v2.md](../examples/migration-v2.md)
- [migration-v1-v2-reference.md](migration-v1-v2-reference.md)
- SvelteKit docs: https://kit.svelte.dev/docs/migrating
