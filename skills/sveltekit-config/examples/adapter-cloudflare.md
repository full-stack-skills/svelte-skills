# adapter-cloudflare

Unified adapter for Cloudflare Workers (Static Assets) and Cloudflare Pages. Supersedes the deprecated `@sveltejs/adapter-cloudflare-workers`.

## 1. Install

```sh
npm i -D @sveltejs/adapter-cloudflare
npm i -D wrangler  # for dev/deploy
```

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-cloudflare';
export default { kit: { adapter: adapter() } };
```

## 2. Wrangler config (Workers with Static Assets)

```jsonc
// wrangler.jsonc
{
  "name": "my-app",
  "main": ".svelte-kit/cloudflare/_worker.js",
  "compatibility_flags": ["nodejs_als"],
  "compatibility_date": "2025-01-01",
  "assets": {
    "binding": "ASSETS",
    "directory": ".svelte-kit/cloudflare"
  }
}
```

## 3. Deploy to Workers

```sh
npm run build
wrangler deploy
```

Set `account_id` if you have multiple accounts.

## 4. Cloudflare Pages (Git integration)

In Cloudflare dashboard → Pages → Create → connect to Git:
- Build command: `npm run build`
- Build output dir: `.svelte-kit/cloudflare`
- Add `nodejs_als` to compatibility flags (Runtime → Compatibility Flags)

## 5. Runtime APIs via `platform`

The `env` object (KV, DO, R2, etc.) is exposed via `event.platform.env`:

```js
// src/app.d.ts
import type { KVNamespace, DurableObjectNamespace } from '@cloudflare/workers-types';
declare global {
  namespace App {
    interface Platform {
      env: {
        MY_KV: KVNamespace;
        MY_DO: DurableObjectNamespace;
      };
    }
  }
}
export {};
```

```js
// +server.js
export async function POST({ request, platform }) {
  const id = platform.env.MY_DO.idFromName('session');
  const stub = platform.env.MY_DO.get(id);
  return stub.fetch(request);
}
```

## 6. Environment variables

Wrangler secrets override static config:

```sh
wrangler secret put MY_SECRET
```

Read in code via `$env/static/private` (preferred — build-time inlining) or `$env/dynamic/private`.

```js
// src/hooks.server.js
import { env } from '$env/dynamic/private';
export async function handle({ event, resolve }) {
  console.log(env.MY_SECRET);
  return resolve(event);
}
```

## 7. Local development with bindings

The `platformProxy` option emulates `platform.env` locally from your wrangler config:

```js
adapter({
  platformProxy: {
    configPath: 'wrangler.jsonc',
    persist: '.wrangler/state'  // keep KV/DO data across restarts
  }
})
```

Run `npm run dev` — `platform.env` will reflect your wrangler bindings.

## 8. Testing the production build locally

```sh
wrangler dev .svelte-kit/cloudflare/_worker.js   # Workers
wrangler pages dev .svelte-kit/cloudflare        # Pages
```

## 9. Node.js compatibility

For libraries that need Node APIs (e.g. `crypto.scryptSync`):

```jsonc
{
  "compatibility_flags": ["nodejs_compat"]
}
```

## 10. `_headers` and `_redirects` for Pages

Place in project root:

```
# _headers
/*
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff

/assets/*
  Cache-Control: public, max-age=31536000, immutable
```

```
# _redirects
/old-page   /new-page   301
/api/*      https://api.example.com/:splat  200
```

These only affect static asset responses. For dynamic routes, return headers from `+server.js` or `handle`.

## 11. Fallback options

```js
adapter({ fallback: 'spa' })   // generate SPA index.html for unknown routes
adapter({ fallback: 'plaintext' }) // return null-body 404 (default for Workers)
```

`spa` mode triggers when `assets.not_found_handling: 'single-page-application'` in wrangler config.

## 12. Migrating from Workers Sites

```diff
-import adapter from '@sveltejs/adapter-cloudflare-workers';
+import adapter from '@sveltejs/adapter-cloudflare';
```

```diff
-  "site": { "bucket": "./.cloudflare/public" }
+  "assets": { "directory": "./.cloudflare/public", "binding": "ASSETS" }
```

## See also

- [Adapter comparison](../references/adapters-comparison.md)
