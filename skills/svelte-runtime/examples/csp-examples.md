# CSP Examples for Svelte 5 SSR

Concrete examples for satisfying a Content Security Policy with Svelte 5's `render()` and `hydratable()` APIs. Covers nonce mode, hash mode, header wiring, and runtime configuration.

> See [`references/csp-reference.md`](../references/csp-reference.md) for the full conceptual reference.

---

## 1. Basic Nonce (Per-Request Dynamic SSR)

Generate a fresh nonce per response and inject it into the SSR pipeline:

```js
// server.js (Express)
import express from 'express';
import { render } from 'svelte/server';
import App from './App.svelte';

const app = express();

app.get('*', async (req, res) => {
  const nonce = crypto.randomUUID();

  const { head, body } = await render(App, {
    props: { url: req.url },
    csp: { nonce }
  });

  res.setHeader(
    'Content-Security-Policy',
    `default-src 'self'; script-src 'self' 'nonce-${nonce}'; style-src 'self' 'nonce-${nonce}'`
  );
  res.send(`<!DOCTYPE html>
<html>
  <head>${head}</head>
  <body>
    <div id="app">${body}</div>
  </body>
</html>`);
});
```

```svelte
<!-- App.svelte -->
<script>
  import { hydratable } from 'svelte';

  // This will produce an inline <script> in head — covered by the nonce
  const config = await hydratable('config', () => ({ featureFlag: true }));
</script>

<h1>Hello, {config.featureFlag ? 'flag-on' : 'flag-off'}</h1>
```

The hydratable bootstrap script ends up with `nonce="<your-nonce>"`, satisfying the CSP.

---

## 2. SvelteKit Hook with Per-Request Nonce

SvelteKit's `handle` hook is the canonical place to mint and propagate a nonce:

```js
// src/hooks.server.js
import crypto from 'node:crypto';

/** @type {import('@sveltejs/kit').Handle} */
export async function handle({ event, resolve }) {
  const nonce = crypto.randomBytes(16).toString('base64');
  event.locals.cspNonce = nonce;

  const response = await resolve(event, {
    transformPageChunk: ({ html }) =>
      html.replaceAll('%sveltekit.nonce%', nonce)
  });

  response.headers.set(
    'Content-Security-Policy',
    [
      `default-src 'self'`,
      `script-src 'self' 'nonce-${nonce}'`,
      `style-src 'self' 'nonce-${nonce}' 'unsafe-hashes'`,
      `img-src 'self' data:`,
      `connect-src 'self'`
    ].join('; ')
  );

  return response;
}
```

```js
// src/routes/+page.server.js
import { render } from 'svelte/server';
import Page from './Page.svelte';

export async function load({ locals, fetch }) {
  const data = await fetch('/api/feed').then(r => r.json());

  const { head, body } = await render(Page, {
    props: { data },
    csp: { nonce: locals.cspNonce }
  });

  return { head, body, data };
}
```

The `%sveltekit.nonce%` token is replaced by SvelteKit on every chunk of HTML it streams, ensuring Svelte's own injected scripts also carry the nonce.

---

## 3. Hash Mode for Static Site Generation

Pre-render at build time and serve the CSP header from the CDN:

```js
// build.js
import { render } from 'svelte/server';
import { writeFileSync, mkdirSync } from 'node:fs';
import App from './App.svelte';
import { routes } from './routes.js';

mkdirSync('dist', { recursive: true });

for (const route of routes) {
  const { head, body, hashes } = await render(App, {
    props: { path: route.path, data: route.data },
    csp: { hash: true }
  });

  const html = `<!DOCTYPE html>
<html>
  <head>
    ${head}
    <meta http-equiv="Content-Security-Policy" content="script-src ${hashes.script.map(h => `'${h}'`).join(' ')}">
  </head>
  <body><div id="app">${body}</div></body>
</html>`;

  writeFileSync(`dist${route.path}/index.html`, html);
}

// Optionally write a separate _headers file for Netlify
writeFileSync('dist/_headers', `/*
  Content-Security-Policy: script-src ${routes[0].hashes.script.map(h => `'${h}'`).join(' ')}
`);
```

```svelte
<!-- App.svelte -->
<script>
  import { hydratable } from 'svelte';
  const data = await hydratable('data', () => loadInitialData());
</script>

<main>{data.title}</main>
```

> Hashes are content-derived — they change whenever the bootstrap script changes. Update the CSP header on every deploy.

---

## 4. Cloudflare Worker / Edge Function with Nonce

```js
// worker.js
export default {
  async fetch(request, env) {
    const nonce = crypto.randomUUID();

    const { head, body } = await render(App, {
      props: { url: request.url },
      csp: { nonce }
    });

    const html = `<!DOCTYPE html>
<html>
  <head>${head}</head>
  <body><div id="app">${body}</div></body>
</html>`;

    return new Response(html, {
      headers: {
        'Content-Security-Policy':
          `default-src 'self'; script-src 'self' 'nonce-${nonce}'`,
        'Content-Type': 'text/html; charset=utf-8'
      }
    });
  }
};
```

---

## 5. CSP Without hydratable() (Nothing to Do)

If your app does not use `hydratable()`, `render()` does **not** emit any inline `<script>` block, and CSP "just works":

```js
import { render } from 'svelte/server';
const { head, body } = await render(App, { props: { foo: 1 } });
// head contains only <style>, <link>, <meta>, etc. — no <script>.
```

You can serve a strict CSP without nonces:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self';
```

Inline event handlers (`<button onclick={...}>`) are compiled by Svelte into JS that runs at hydration time — they are not parsed from inline `<script>` tags, so they do not need a nonce.

---

## 6. Diagnosing CSP Failures

If your hydratable data shows up as `undefined` after hydration, CSP is the first thing to check:

```js
// After hydration, force any pending errors to surface
import { hydrate, flushSync } from 'svelte';
import App from './App.svelte';

try {
  const app = hydrate(App, { target: document.querySelector('#app') });
  flushSync();
} catch (err) {
  console.error('Hydration failed:', err);
}
```

A blocked inline script shows up in DevTools Console as:

```
Refused to execute inline script because it violates the following
Content Security Policy directive: "script-src 'self' 'nonce-abc123'".
```

Fix by ensuring the nonce in your `render()` call matches the one in the response header, or that your hash list is up to date.

---

## 7. CSP and Streaming SSR

Nonce mode plays well with streaming — each chunk can carry the nonce attribute, and the browser evaluates the inline script when it sees it. Hash mode is fragile with streaming because the hash of a script is only known after the script body is fully rendered; Svelte currently does not support streaming with `csp.hash: true`, and Svelte's docs warn that hash mode will "interfere with streaming SSR in the future."

If you anticipate streaming (TTFB-sensitive apps, large pages), **prefer nonce mode** even when static generation is tempting.

---

## 8. Runtime / build-time detection

If you want to enable CSP only in production, branch on `NODE_ENV` or your build target:

```js
import { render } from 'svelte/server';
import App from './App.svelte';

const isProd = process.env.NODE_ENV === 'production';
const nonce = isProd ? crypto.randomUUID() : undefined;

const { head, body } = await render(App, {
  props,
  csp: nonce ? { nonce } : undefined
});

const headers = new Headers({ 'Content-Type': 'text/html' });
if (nonce) {
  headers.set(
    'Content-Security-Policy',
    `default-src 'self'; script-src 'self' 'nonce-${nonce}'`
  );
}

return new Response(html({ head, body }), { headers });
```

In development, omit the CSP header entirely — Vite's HMR injects inline scripts that would otherwise be blocked.

---

## Summary

| Mode | When | Source of trust |
|---|---|---|
| `csp.nonce` | per-request dynamic SSR | matching nonce in response header |
| `csp.hash: true` | static site generation | matching hashes in response header |
| omitted | no `hydratable()` in the tree | no inline `<script>` emitted |

For the deep dive see [`references/csp-reference.md`](../references/csp-reference.md).