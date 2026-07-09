# Adapters comparison

SvelteKit adapters transform the built app into a deployable format for a specific platform.

## When to use which

| Adapter | When | Pros | Cons |
|---------|------|------|------|
| `@sveltejs/adapter-auto` | Prototyping, undefined target | Picks the right adapter for known envs | Can't pass platform-specific options |
| `@sveltejs/adapter-node` | Node server (VPS, Docker, Heroku, Fly, Render) | Full Node API access, custom servers | You manage hosting/scaling |
| `@sveltejs/adapter-static` | SSG / static host / SPA fallback | Cheapest, fastest TTFB, CDN-friendly | No SSR for dynamic routes (unless prerendered) |
| `@sveltejs/adapter-cloudflare` | Cloudflare Workers + Pages | Edge, generous free tier, KV/DO/R2 bindings | 1MB worker size limit (3MB paid), no `fs` |
| `@sveltejs/adapter-netlify` | Netlify Functions or Edge | Built-in forms, identity, edge functions | Edge = Deno, less Node support |
| `@sveltejs/adapter-vercel` | Vercel | Image opt, ISR, skew protection | Pricing at scale |

## Zero-config (via `adapter-auto`)

Detects environment and installs/picks the right adapter:
- Cloudflare Pages → `adapter-cloudflare`
- Netlify → `adapter-netlify`
- Vercel → `adapter-vercel`
- Azure Static Web Apps → `svelte-adapter-azure-swa`
- SST (AWS) → `svelte-kit-sst`
- Google Cloud Run → `adapter-node`

Once you settle on a target, install that adapter explicitly so it lands in the lockfile and you can pass options.

## Adapter options at a glance

### `adapter-node`

```js
adapter({
  out: 'build',           // output directory
  precompress: true,      // gzip + brotli
  envPrefix: ''           // prefix for env vars (e.g. 'MY_APP_')
})
```

Env vars: `PORT`, `HOST`, `SOCKET_PATH`, `ORIGIN`, `PROTOCOL_HEADER`, `HOST_HEADER`, `PORT_HEADER`, `ADDRESS_HEADER`, `XFF_DEPTH`, `BODY_SIZE_LIMIT`, `SHUTDOWN_TIMEOUT`, `IDLE_TIMEOUT`, `KEEP_ALIVE_TIMEOUT`, `HEADERS_TIMEOUT`, `LISTEN_PID`, `LISTEN_FDS`.

### `adapter-static`

```js
adapter({
  pages: 'build',
  assets: 'build',
  fallback: undefined,    // '200.html' for SPA mode
  precompress: false,
  strict: true            // error if some routes aren't prerendered
})
```

### `adapter-cloudflare`

```js
adapter({
  config: undefined,                  // path to wrangler.jsonc
  platformProxy: { configPath, environment, persist },
  fallback: 'plaintext',              // or 'spa'
  routes: { include: ['/*'], exclude: ['<all>'] }
})
```

### `adapter-netlify`

```js
adapter({
  edge: false,    // true for Deno-based edge functions
  split: false    // true for per-route functions (not with edge: true)
})
```

### `adapter-vercel`

```js
adapter({
  runtime: 'nodejs22.x',     // or 'edge'
  regions: ['iad1'],
  memory: 1024,
  maxDuration: 10,
  isr: { expiration: 60 },
  images: { sizes: [], formats: [], minimumCacheTTL: 0, domains: [] },
  split: false
})
```

Per-route override via `export const config` in `+server.js` or `+page.js`.

## Platform-specific context

Some adapters expose platform info via `event.platform`:

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface Platform {
      env?: Record<string, unknown>;
    }
  }
}
export {};
```

- Cloudflare: `env`, `ctx`, `caches`, `cf`
- Netlify: `context` (Netlify Functions/Edge context)
- Vercel: limited — use `$env` for env vars

Always prefer `$env/static/private` for env vars.

## Writing custom adapters

If no official adapter fits, write your own using the builder API. See `examples/writing-adapter.md` for the full pattern.

The minimum adapter:
1. `name`, `adapt(builder)` are required
2. Inside `adapt`: write client, server, prerendered; emit a manifest
3. Bundle and ship

## Migrating between adapters

```sh
npm rm @sveltejs/adapter-X
npm i -D @sveltejs/adapter-Y
```

Then update `svelte.config.js`. No code changes needed if you weren't using platform-specific features.

## Cross-platform caveats

- **`fs` not available** on edge: use `read` from `$app/server`
- **Cold starts**: serverless (Netlify Functions, Vercel Functions) — edge and Cloudflare Workers are usually faster
- **Long requests**: serverless have time limits (10s Hobby / 900s Enterprise on Vercel)
- **Streaming**: supported on all adapters but Node streams are unique

## See also

- [examples/build-preview.md](../examples/build-preview.md)
- [examples/adapter-node.md](../examples/adapter-node.md)
- [examples/adapter-static.md](../examples/adapter-static.md)
- [examples/adapter-cloudflare.md](../examples/adapter-cloudflare.md)
- [examples/adapter-netlify.md](../examples/adapter-netlify.md)
- [examples/adapter-vercel.md](../examples/adapter-vercel.md)
- [examples/writing-adapter.md](../examples/writing-adapter.md)
