# Project Types Examples

Each project type corresponds to an adapter and a config. These snippets show the minimum needed for each.

## 1. Default (SSR + CSR hybrid)

Default behavior. SSR the first paint, CSR subsequent navigation.

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-auto';
/** @type {import('@sveltejs/kit').Config} */
export default { kit: { adapter: adapter() } };
```

## 2. Static site generation (SSG)

```js
import adapter from '@sveltejs/adapter-static';
/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    adapter: adapter({ pages: 'build', assets: 'build' }),
    prerender: { entries: [] }
  }
};
```

```js
// src/routes/+layout.js
export const prerender = true;  // apply to all routes
```

## 3. Single-page app (SPA)

```js
import adapter from '@sveltejs/adapter-static';
/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    adapter: adapter({
      pages: 'build',
      assets: 'build',
      fallback: 'index.html'   // SPA fallback
    })
  }
};
```

```js
// src/routes/+layout.js
export const ssr = false;  // no SSR; client renders everything
```

## 4. Multi-page app (MPA)

```svelte
<!-- src/routes/+layout.svelte -->
<body data-sveltekit-reload>
  {@render children()}
</body>
```

```js
// src/routes/+layout.js
export const csr = false;  // remove all JS
```

## 5. Separate backend (Go/Rust/PHP/etc.)

```js
import adapter from '@sveltejs/adapter-node';
/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    adapter: adapter(),
    // Frontend calls backend directly via fetch
  }
};
```

```js
// src/routes/+page.server.js
/** @type {import('./$types').PageServerLoad} */
export async function load({ fetch }) {
  const res = await fetch('https://api.example.com/items');
  return { items: await res.json() };
}
```

## 6. Serverless on Vercel

```sh
npm install -D @sveltejs/adapter-vercel
```

```js
import adapter from '@sveltejs/adapter-vercel';
export default {
  kit: {
    adapter: adapter({
      runtime: 'edge'  // or 'nodejs'
    })
  }
};
```

## 7. Serverless on Cloudflare

```sh
npm install -D @sveltejs/adapter-cloudflare
```

```js
import adapter from '@sveltejs/adapter-cloudflare';
export default {
  kit: {
    adapter: adapter({
      routes: { include: ['/*'], exclude: ['<all>'] }
    })
  }
};
```

## 8. Self-hosted Node server

```sh
npm install -D @sveltejs/adapter-node
```

```js
import adapter from '@sveltejs/adapter-node';
export default {
  kit: { adapter: adapter({ out: 'build' }) }
};
```

```sh
node build   # runs the Node server
```

## 9. Container (Docker)

```dockerfile
# Dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY build ./build
EXPOSE 3000
CMD ["node", "build"]
```

```js
// svelte.config.js — same as adapter-node
import adapter from '@sveltejs/adapter-node';
export default { kit: { adapter: adapter() } };
```

## 10. Library (publishable Svelte components)

```sh
npx sv create my-lib --template library
```

```js
// svelte.config.js (generated)
import adapter from '@sveltejs/package';
export default {
  kit: {
    adapter: adapter({ sourcemap: true })
  }
};
```

## 11. Mobile (Tauri / Capacitor)

```js
import adapter from '@sveltejs/adapter-static';
export default {
  kit: {
    adapter: adapter({
      pages: 'build',
      assets: 'build',
      fallback: 'index.html'
    })
  },
  output: { bundleStrategy: 'single' }   // reduce concurrent requests
};
```

Then wrap with Tauri or Capacitor — see their respective docs.

## 12. Embedded device (single bundle)

```js
// svelte.config.js
export default {
  kit: { adapter: adapter() },
  output: { bundleStrategy: 'single' }
};
```

Use when the runtime limits concurrent HTTP connections (TVs, microcontrollers).