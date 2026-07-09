# Migration v1 → v2

Run `npx sv migrate sveltekit-2` to auto-fix most breaking changes. Below are the changes it handles and ones it can't.

## 1. `error` and `redirect` are no longer thrown

```diff
-throw error(404, { message: 'Not found' });
+error(404, { message: 'Not found' });

-throw redirect(303, '/login');
+redirect(303, '/login');
```

If you catch the error in a `try`, use `isHttpError` / `isRedirect` from `@sveltejs/kit` to distinguish expected from unexpected.

## 2. Cookies require `path`

```diff
-cookies.set('session', value);
+cookies.set('session', value, { path: '/' });
```

Applies to `cookies.set`, `cookies.delete`, `cookies.serialize`. Most cases want `path: '/'`.

## 3. Top-level promises in `load` are NOT awaited

```diff
-// v1: top-level promises were auto-awaited
-export function load({ fetch }) {
-  return {
-    user: fetch('/api/me').then(r => r.json()),
-    settings: fetch('/api/settings').then(r => r.json())
-  };
-}

+// v2: must await explicitly
+export async function load({ fetch }) {
+  const [user, settings] = await Promise.all([
+    fetch('/api/me').then(r => r.json()),
+    fetch('/api/settings').then(r => r.json())
+  ]);
+  return { user, settings };
+}
```

This change unblocks streaming.

## 4. `goto` no longer accepts external URLs

```diff
-goto('https://example.com');
+window.location.href = 'https://example.com';
```

`state` object now determines `$page.state` and must conform to `App.PageState`.

## 5. Paths are consistently relative (or absolute)

Default `paths.relative = true`. `%sveltekit.assets%`, `base`, `assets` from `$app/paths` all behave consistently.

If you need absolute paths (rare), set `paths.relative = false` in `svelte.config.js`.

## 6. `preloadCode` requires `base` prefix

```diff
-preloadCode('/about');      // v1: did not need base
+preloadCode(base + '/about');  // v2: include base

-preloadData('/about');      // unchanged
+preloadData(base + '/about');
```

`preloadCode` now takes a single argument, not multiple.

## 7. `resolvePath` → `resolveRoute`

```diff
-import { resolvePath } from '@sveltejs/kit';
-import { base } from '$app/paths';
-const path = base + resolvePath('/blog/[slug]', { slug });
+import { resolveRoute } from '$app/paths';
+const path = resolveRoute('/blog/[slug]', { slug });
```

`resolveRoute` handles `base` internally.

## 8. `$app/stores` deprecated (2.12+)

Migrate to `$app/state`:

```diff
-import { page } from '$app/stores';
+import { page } from '$app/state';

-{$page.data.title}
+{page.data.title}
```

Requires Svelte 5. Run `npx sv migrate app-state` to auto-convert.

## 9. Form file inputs require `enctype`

```svelte
<!-- Form with file input MUST have enctype -->
<form method="POST" enctype="multipart/form-data" use:enhance>
  <input type="file" name="avatar" />
</form>
```

SvelteKit 2 throws an error during `use:enhance` if a file input is present without `enctype="multipart/form-data"`.

## 10. `use:enhance` callbacks — `form` and `data` removed

```diff
-enhance(({ form, data, ...rest }) => {
+enhance(({ formElement, formData, ...rest }) => {
```

The deprecated `form` (HTMLFormElement) and `data` (FormData) were removed. Use `formElement` and `formData`.

## 11. `getRequest` no longer throws early

In SvelteKit 2, `getRequest` from `@sveltejs/kit/node` returns even for oversized requests. The error fires only when the body is read. Improves diagnostics.

## 12. `vitePreprocess` no longer re-exported

```diff
-import { vitePreprocess } from '@sveltejs/kit/vite';
+import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';
```

`@sveltejs/vite-plugin-svelte` is now a peer dependency.

## 13. Updated peer deps

SvelteKit 2 requires:
- Node 18.13+
- `svelte@4` (5 recommended)
- `vite@5`
- `typescript@5`
- `@sveltejs/vite-plugin-svelte@3`
- `@sveltejs/adapter-cloudflare@3`
- `@sveltejs/adapter-netlify@3`
- `@sveltejs/adapter-node@2`
- `@sveltejs/adapter-static@3`
- `@sveltejs/adapter-vercel@4`

`npx sv migrate sveltekit-2` updates these for you.

## 14. tsconfig changes

- `moduleResolution: 'bundler'` (instead of 'node')
- `verbatimModuleSyntax: true` (replaces `importsNotUsedAsValues` and `preserveValueImports` — remove those if present)
- Don't use `paths` or `baseUrl` in tsconfig — use `alias` in `svelte.config.js` instead

## 15. Improved `handleError` signature

```ts
export const handleError: HandleServerError = ({ error, event, status, message }) => {
  // status and message are new in v2
  // status defaults to 500, message to 'Internal Error'
};
```

## See also

- [Migration v1→v2 reference](../references/migration-v1-v2-reference.md)
