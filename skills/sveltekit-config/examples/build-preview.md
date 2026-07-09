# Build & Preview

`vite build` runs Vite's optimization + the chosen adapter. `vite preview` serves the build locally in Node.

## 1. Default build

```sh
npm run build        # = vite build
```

Output goes to wherever the adapter writes (default `build/` for `adapter-node`).

## 2. Custom build output directory

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-node';
export default {
  kit: {
    adapter: adapter({ out: 'dist' })
  }
};
```

## 3. Preview the build

```sh
npm run preview      # = vite preview, defaults to port 4173
```

Preview runs the **adapter-node** server in Node. It does NOT emulate Cloudflare/Netlify/Vercel-specific behavior (`event.platform`, edge functions, etc.).

## 4. Build-time env vars

Vars prefixed `VITE_` are inlined at build time and available client-side:

```sh
VITE_ANALYTICS_ID=UA-12345 npm run build
```

```js
import { VITE_ANALYTICS_ID } from '$env/static/private';
// or for client:
import.meta.env.VITE_ANALYTICS_ID;
```

For server-only build-time vars, use `$env/static/private` (imported from `@sveltejs/kit`). Values are replaced at build time — no runtime lookup.

## 5. Skip prerendering during dev

`prerender = true` only runs at build time, but if your `load` does expensive work, guard it:

```js
import { building } from '$app/environment';

export async function load() {
  if (!building) {
    // skip heavy work in dev
  }
}
```

## 6. Inspect the build

```sh
npx vite build --mode development  # unminified, easier to read
```

Then browse `build/_app/immutable/`. Use `rollup-plugin-visualizer` for a treemap:

```js
// vite.config.js
import { visualizer } from 'rollup-plugin-visualizer';
export default {
  plugins: [sveltekit(), visualizer({ filename: 'stats.html' })]
};
```

## 7. Build env files

Vite loads `.env`, `.env.local`, `.env.[mode]`, `.env.[mode].local` automatically. `.env` is for shared defaults; `.env.local` for personal overrides (gitignored).

```sh
.env                # shared, all envs
.env.local          # personal, gitignored
.env.production     # production-only
.env.development    # dev-only
```

## 8. Build-time database init skip

```js
import { building } from '$app/environment';
import { initDb } from '$lib/server/db';

if (!building) await initDb();
```

Required because SvelteKit loads `+page/layout(.server).js` for analysis during the build.

## 9. Build for multiple targets

```sh
# svelte.config.js: read adapter from env or argv
import node from '@sveltejs/adapter-node';
import vercel from '@sveltejs/adapter-vercel';

const target = process.env.TARGET;
const adapter = target === 'vercel' ? vercel : node;

export default { kit: { adapter: adapter() } };
```

```sh
TARGET=vercel vite build && mv .vercel/output build-vercel
TARGET=node vite build && mv build build-node
```

## 10. Build for preview without serving

If you just want to inspect the static output, `vite build` then `ls build/` or open `build/index.html` directly. No preview server needed for fully-prerendered sites.

## See also

- [Building reference](../references/building-reference.md)
- [Adapter comparison](../references/adapters-comparison.md)
