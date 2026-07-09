# adapter-vercel

Serverless or edge functions, ISR, native image optimization, skew protection.

## 1. Install

```sh
npm i -D @sveltejs/adapter-vercel
```

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-vercel';
export default { kit: { adapter: adapter() } };
```

Push to a Git repo connected to Vercel — auto-deploys on push.

## 2. Image optimization

```js
// svelte.config.js
adapter({
  images: {
    sizes: [640, 828, 1200, 1920, 3840],
    formats: ['image/avif', 'image/webp'],
    minimumCacheTTL: 300,
    domains: ['example-app.vercel.app']
  }
})
```

Vercel's Image Optimization pipeline resizes/transforms at the edge. Configure remote domains to allow them.

## 3. ISR (Incremental Static Regeneration)

```js
// src/routes/blog/[slug]/+page.js
import { BYPASS_TOKEN } from '$env/static/private';

export const config = {
  isr: {
    expiration: 60,           // revalidate every 60s
    bypassToken: BYPASS_TOKEN,
    allowQuery: ['search']
  }
};
export const prerender = false;
```

Generate `BYPASS_TOKEN` (≥32 chars):

```sh
node -e "console.log(crypto.randomUUID())"
```

Add as env var in Vercel dashboard. To force revalidate:

```sh
curl -H "x-prerender-revalidate: $BYPASS_TOKEN" https://my-site.vercel.app/
```

ISR has no effect on routes with `prerender = true`.

## 4. Runtime per route

```js
/** @type {import('@sveltejs/adapter-vercel').Config} */
export const config = {
  runtime: 'edge',         // or 'nodejs20.x', 'nodejs22.x'
  regions: ['iad1', 'fra1'],  // edge regions for edge runtime
  split: true              // deploy this route as its own function
};
```

## 5. Environment variables

Set in Vercel dashboard or via CLI:

```sh
vercel env pull .env.development.local
```

Access Vercel's deployment vars via `$env/static/private`:

```js
import { VERCEL_COMMIT_REF } from '$env/static/private';
```

## 6. Skew protection

Enable in Project Settings → Advanced → Skew Protection. Routes all client requests to their original deployment until a fresh load. Cookies are used; multi-tab users on older tabs will fall back to SvelteKit's built-in skew detection.

## 7. Per-route function memory & timeout

```js
export const config = {
  memory: 3008,    // MB, max 3008 on Pro/Enterprise
  maxDuration: 60  // seconds, max 900 on Enterprise
};
```

Hobby defaults: 1024 MB memory, 10s timeout.

## 8. Vercel utilities (`waitUntil`)

```sh
npm i @vercel/functions
```

```js
import { waitUntil } from '@vercel/functions';
export async function POST({ request }) {
  waitUntil(logAsync(request));
  return new Response('ok');
}
```

## 9. Deployment protection + `read()` from `$app/server`

If using `read()` in edge functions AND Deployment Protection is enabled, also enable [Protection Bypass for Automation](https://vercel.com/docs/deployment-protection/methods-to-bypass-deployment-protection/protection-bypass-automation) or `read()` will get 401.

## 10. No `/api` route collisions

Vercel's `/api` directory is NOT included if your SvelteKit app has `/api/*` routes. Use `+server.js` files instead of standalone Vercel Functions.

## See also

- [Adapter comparison](../references/adapters-comparison.md)
