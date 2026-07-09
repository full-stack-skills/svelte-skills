# adapter-static

Static site generation (SSG) or SPA fallback. Use for fully prerendered sites or hybrid static+SPA.

## 1. Install & configure (full SSG)

```sh
npm i -D @sveltejs/adapter-static
```

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-static';
export default { kit: { adapter: adapter() } };
```

```js
// src/routes/+layout.js
export const prerender = true;
```

All pages must be prerenderable (no dynamic data per-user). The adapter will error if any route isn't.

## 2. Build & deploy

```sh
npm run build
# uploads build/ to any static host (Netlify, S3, GitHub Pages, etc.)
```

## 3. SPA fallback

Single fallback HTML handles any non-prerendered URL:

```js
adapter({ fallback: '200.html' })
```

```js
// src/routes/+layout.js
export const ssr = false;       // disable SSR for non-prerendered routes
export const prerender = true;  // but still prerender where possible
```

Avoid `index.html` for fallback — conflicts with prerendered homepage.

## 4. Per-route prerender override

Prerender the homepage only:

```js
// src/routes/+page.js
export const prerender = true;
```

Disable prerendering for a specific page:

```js
// src/routes/blog/[slug]/+page.js
export const prerender = false;
```

## 5. Custom output directory

```js
adapter({ pages: 'public', assets: 'public' })
```

Both must match if you want them merged; rare to need different dirs.

## 6. GitHub Pages

For a repo at `username.github.io/my-repo`:

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-static';

export default {
  kit: {
    adapter: adapter({ fallback: '404.html' }),
    paths: { base: process.env.BASE_PATH || '' }
  }
};
```

Add `.nojekyll` to `static/` (empty file) to bypass Jekyll. GitHub Actions workflow:

```yaml
# .github/workflows/deploy.yml
name: Deploy to GitHub Pages
on: { push: { branches: ['main'] } }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v6
        with: { node-version: 20, cache: npm }
      - run: npm i
      - run: npm run build
        env: { BASE_PATH: '/my-repo' }
      - uses: actions/upload-pages-artifact@v5
        with: { path: build/ }
  deploy:
    needs: build
    runs-on: ubuntu-latest
    permissions: { pages: write, id-token: write }
    environment: { name: github-pages, url: '${{steps.deployment.outputs.page_url}}' }
    steps:
      - uses: actions/deploy-pages@v5
```

## 7. precompress

```js
adapter({ precompress: true })
```

Generates `.br` and `.gz` versions of every HTML/asset. Configure your host (e.g. nginx, Caddy) to serve them when supported.

## 8. Disable strict check

By default `adapter-static` errors if any route isn't prerendered AND no `fallback` is set:

```js
adapter({ strict: false })
```

Use only when you know some pages will be handled elsewhere (e.g. a CDN rule).

## 9. Vercel zero-config

On Vercel, omit adapter options entirely:

```js
adapter()   // no options — Vercel configures automatically
```

## 10. Dynamic routes via entries

For dynamic routes like `[slug]`, prerender a list of known entries by exporting `entries` from `+page.js`:

```js
// src/routes/blog/[slug]/+page.js
export async function entries() {
  const posts = await getAllPosts();
  return posts.map(p => ({ slug: p.slug }));
}
export const prerender = true;
```

Without `entries`, only the first match may be prerendered (depending on adapter version) — always provide `entries` for dynamic routes.

## See also

- [Adapter comparison](../references/adapters-comparison.md)
