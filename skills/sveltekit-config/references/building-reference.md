# Building reference

How `vite build` and `vite preview` work, env vars, build artifacts.

## Two-stage build

1. **Vite**: optimizes server code, browser code, service worker. Runs prerendering if applicable.
2. **Adapter**: takes the production build and tailors it for the target platform.

`vite build` runs both stages. Usually invoked via `npm run build`.

## During the build

SvelteKit loads your `+page/layout(.server).js` files for analysis. Any code that should NOT execute at this stage must check `building`:

```js
import { building } from '$app/environment';
if (!building) {
  // one-time init code (database, etc.)
}
```

This applies to:
- `init` hooks
- Module-level DB connections
- File system reads of dynamic paths

## Preview

```sh
npm run preview    # vite preview
```

Runs the production build via `adapter-node` in Node. NOT a perfect replica of your deployed app:

| Feature | `vite preview` | Real platform |
|---------|---------------|---------------|
| HTTP server | Node | Platform-specific |
| `event.platform` | Empty/undefined | Real bindings (KV, DO, etc.) |
| Edge functions | None | Yes (on edge platforms) |
| ISR | None | Yes (Vercel) |
| Skew protection | None | Yes (Vercel/Netlify) |

For accurate local testing, use the platform CLI: `wrangler dev`, `netlify dev`, `vercel dev`.

## Env vars

Vite's standard env file loading (in order of precedence):

1. `.env.[mode].local`
2. `.env.[mode]`
3. `.env.local`
4. `.env`

`mode` is `development` for `vite dev`, `production` for `vite build`/`vite preview`.

### Public env vars (inlined)

Vars prefixed `VITE_` are replaced at build time and exposed to client code:

```js
// src/lib/config.js
export const ANALYTICS_ID = import.meta.env.VITE_ANALYTICS_ID;
```

For SvelteKit-specific access, use `$env/static/public` or `$env/dynamic/public`.

### Private env vars

Use `$env/static/private` (build-time inlined) or `$env/dynamic/private` (runtime lookup).

In **dev** and **preview**, SvelteKit reads `.env` automatically.
In **production**, `.env` is NOT loaded. Options:

- `node --env-file=.env build` (Node 20.6+)
- `node -r dotenv/config build`
- Set vars via your platform's UI/CLI

## Prerendering

Configured per-route or globally:

```js
// svelte.config.js
export default {
  kit: {
    prerender: {
      handleHttpError: 'warn',  // or 'fail', or ({ path, message }) => boolean
      handleMissingId: 'warn',
      entries: ['*']
    }
  }
};
```

Per-route:

```js
// +page.js
export const prerender = true;
```

Or via `export const entries` for dynamic routes:

```js
export function entries() {
  return [{ slug: 'hello' }, { slug: 'world' }];
}
```

## Inspect the build

```sh
npm run build -- --mode development  # unminified output
```

Browse `build/` (or wherever the adapter writes).

For bundle analysis:

```js
// vite.config.js
import { visualizer } from 'rollup-plugin-visualizer';
export default {
  plugins: [sveltekit(), visualizer({ filename: 'stats.html' })]
};
```

## Build artifacts

- `build/_app/immutable/` — hashed JS/CSS chunks
- `build/_app/version.json` — current build version (for skew detection)
- `build/index.html` — homepage HTML (if prerendered)
- `build/server/` — server entry (only for server adapters)
- `build/handler.js` — custom server middleware (adapter-node only)
- `build/index.js` — main server entry (adapter-node only)

## Skew detection

After build, SvelteKit's runtime compares the build version with the server's. If they don't match (new deploy), it triggers a hard reload. Override:

```js
import { updated } from '$app/state';
$effect(() => {
  if (updated.current) console.log('New version available');
});
```

Platform skew protection (Vercel, Netlify) routes requests to the original deployment until a fresh load.

## Common build errors

- **`Cannot prerender ...`** — dynamic route needs `entries()` or has unprerenderable content
- **`hydration mismatch`** — SSR output differs from client render. Use `browser` checks to guard client-only code.
- **`build is too large`** — see `examples/performance.md` for code-splitting.
- **`Module not found`** during prerender — usually a missing env var; check `$env/static/private` usage.

## See also

- [examples/build-preview.md](../examples/build-preview.md)
- [examples/adapter-node.md](../examples/adapter-node.md)
