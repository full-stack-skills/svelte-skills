# Migration v1 → v2 reference

Full list of breaking changes from SvelteKit 1 to 2.

Run `npx sv migrate sveltekit-2` first to auto-fix most. Then manually verify.

## Prerequisites

- Node 18.13+
- Svelte 4 (5 recommended)
- TypeScript 5
- Vite 5
- `@sveltejs/vite-plugin-svelte@3`

We recommend upgrading to the latest SvelteKit 1.x first to get deprecation warnings.

## Breaking changes

### 1. `error` and `redirect` no longer thrown by user

```diff
-throw error(404, { message: 'Not found' });
+error(404, { message: 'Not found' });
```

`svelte-migrate` handles this. If you catch errors in a `try` block, use `isHttpError` / `isRedirect`:

```js
import { isHttpError, isRedirect } from '@sveltejs/kit';

try {
  // ...
} catch (e) {
  if (isHttpError(e)) { /* expected */ }
  else if (isRedirect(e)) { /* expected */ }
  else { throw e; }
}
```

### 2. Cookies require `path`

```diff
-cookies.set('session', value);
+cookies.set('session', value, { path: '/' });
```

Most cases want `path: '/'`. `''` = current path, `'.'` = current directory.

`svelte-migrate` adds comments — manual fix needed.

### 3. Top-level promises not auto-awaited in `load`

```diff
-export function load({ fetch }) {
-  return {
-    user: fetch('/api/me').then(r => r.json())
-  };
-}

+export async function load({ fetch }) {
+  const user = await fetch('/api/me').then(r => r.json());
+  return { user };
+}
```

For multiple:

```js
const [a, b] = await Promise.all([
  fetch(url1).then(r => r.json()),
  fetch(url2).then(r => r.json())
]);
```

### 4. `goto` rejects external URLs

```diff
-goto('https://example.com');
+window.location.href = 'https://example.com';
```

The `state` object must conform to `App.PageState`.

### 5. Paths consistently relative

Default `paths.relative = true`. `%sveltekit.assets%`, `base`, `assets` from `$app/paths` all behave consistently.

```js
// svelte.config.js
export default {
  kit: { paths: { relative: false } }  // if you need absolute paths
};
```

### 6. Server fetches not trackable

`dangerZone.trackServerFetches` removed. Server `fetch` URLs no longer trigger load reruns (was a privacy risk).

### 7. `preloadCode` requires `base` prefix

```diff
-preloadCode('/about');
+preloadCode(base + '/about');
```

`preloadData` was already prefixed. Now both are consistent.

`preloadCode` takes a single argument (no more variadic).

### 8. `resolvePath` → `resolveRoute`

```diff
-import { resolvePath } from '@sveltejs/kit';
-const path = base + resolvePath('/blog/[slug]', { slug });
+import { resolveRoute } from '$app/paths';
+const path = resolveRoute('/blog/[slug]', { slug });
```

`resolveRoute` handles `base` internally.

### 9. `handleError` gets `status` and `message`

```ts
export const handleError: HandleServerError = ({ error, event, status, message }) => {
  // status: number (500 by default)
  // message: string ('Internal Error' by default)
};
```

Errors now consistently go through `handleError`.

### 10. Dynamic env vars not allowed during prerendering

Use `$env/static/*` during prerendering. `$env/dynamic/*` cannot be used at build time. At runtime, `$env/dynamic/public` is fetched from `/_app/env.js` rather than baked into HTML.

### 11. `use:enhance` callbacks: `form` / `data` removed

```diff
-enhance(({ form, data, action, cancel, submitter }) => {
+enhance(({ formElement, formData, action, cancel, submitter }) => {
```

Deprecated for a long time; now removed.

### 12. Form file inputs must use `multipart/form-data`

```svelte
<form method="POST" enctype="multipart/form-data" use:enhance>
  <input type="file" name="avatar" />
</form>
```

Without `enctype`, JS submissions throw an error. Non-JS submissions silently lose the file.

### 13. Generated `tsconfig.json` is more strict

- `moduleResolution: 'bundler'` (recommended by TS)
- `verbatimModuleSyntax: true` (replaces `importsNotUsedAsValues` + `preserveValueImports`)
- Don't use `paths` or `baseUrl` — use `alias` in `svelte.config.js`

### 14. `getRequest` doesn't throw early

`getRequest` from `@sveltejs/kit/node` returns even for oversized requests. Errors fire when body is read. Better diagnostics.

### 15. `vitePreprocess` no longer re-exported

```diff
-import { vitePreprocess } from '@sveltejs/kit/vite';
+import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';
```

### 16. `$app/stores` deprecated (since 2.12)

Migrate to `$app/state` (requires Svelte 5):

```diff
-import { page } from '$app/stores';
+import { page } from '$app/state';

-{$page.data.title}
+{page.data.title}
```

Run `npx sv migrate app-state` to auto-convert most.

### 17. Peer dependency changes

SvelteKit 2 requires:
- Node 18.13+
- svelte@4
- vite@5
- typescript@5
- @sveltejs/vite-plugin-svelte@3 (peerDep now)
- @sveltejs/adapter-cloudflare@3
- @sveltejs/adapter-cloudflare-workers@2
- @sveltejs/adapter-netlify@3
- @sveltejs/adapter-node@2
- @sveltejs/adapter-static@3
- @sveltejs/adapter-vercel@4

## Step-by-step migration

1. Update to latest SvelteKit 1.x
2. Address deprecation warnings
3. Update to Svelte 4 (and ideally 5): `npm i svelte@4` then `npm i svelte@5`
4. Run `npx sv migrate sveltekit-2`
5. Update peer deps (the migrator does this)
6. Review changes; manually fix cookie paths and any patterns the migrator couldn't
7. Test thoroughly — especially forms, load functions, hooks
8. Update to Svelte 5 if desired: `npx sv migrate svelte-5`
9. Run `npx sv migrate app-state` for `$app/stores` → `$app/state`

## See also

- [examples/migration-v2.md](../examples/migration-v2.md)
- SvelteKit 2 release notes
