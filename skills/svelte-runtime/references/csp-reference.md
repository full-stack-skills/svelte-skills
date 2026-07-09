# CSP (Content Security Policy) Reference for Svelte 5 SSR

`render()` from `svelte/server` can emit an inline `<script>` block — most notably when you use [`hydratable()`](./ssr-reference.md) — and that block needs to satisfy your CSP. This reference covers the two strategies Svelte supports: **nonce** and **hash**.

---

## 1. Why CSP Matters in Svelte SSR

A typical `Content-Security-Policy` header might look like:

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123'; style-src 'self' 'nonce-abc123';
```

When `hydratable()` is used inside a component, `render()` injects a small bootstrap script that replays the serialised data during hydration. Browsers will refuse to execute this script unless it is allowed by the CSP:

- with a matching `'nonce-…'` value, **or**
- with a matching `'sha256-…'` / `'sha384-…'` / `'sha512-…'` hash.

Svelte supports both mechanisms via the `csp` option.

---

## 2. Nonce Mode

```js
import { render } from 'svelte/server';
import App from './App.svelte';

const nonce = crypto.randomUUID();

const { head, body } = await render(App, {
  csp: { nonce }
});
```

What you get:

- `head` contains an inline `<script nonce="…">…</script>` block (the hydratable bootstrap).
- The same nonce must appear in the response's `Content-Security-Policy` header.

```js
const response = new Response();
response.headers.set(
  'Content-Security-Policy',
  `script-src 'nonce-${nonce}'`
);
```

> **"Nonce" = number used once.** Generate a fresh value for **every** response. Reusing a nonce across responses weakens its security guarantee.

### When to use nonce

- Per-request dynamic SSR (SvelteKit `+page.server.js`, Express, Hono, etc.).
- Any deployment where you can inject a per-request response header.

### Generating a strong nonce

```js
// Node 18+
const nonce = crypto.randomUUID();

// Or with crypto.randomBytes
import crypto from 'node:crypto';
const nonce = crypto.randomBytes(16).toString('base64');
```

Avoid `Math.random()`-based nonces — they're guessable.

---

## 3. Hash Mode

If you pre-render HTML at build time (no per-request header possible), use hashes:

```js
import { render } from 'svelte/server';
import App from './App.svelte';

const { head, body, hashes } = await render(App, {
  csp: { hash: true }
});
```

What you get:

- `hashes.script` is `string[]` — entries like `'sha256-AbCdEf123…'`.
- The hash already includes the algorithm prefix, so you can drop the entries straight into the CSP header.

```js
const response = new Response(renderedStream, {
  headers: {
    'Content-Security-Policy':
      `script-src ${hashes.script.map(h => `'${h}'`).join(' ')}`
  }
});
```

### When to use hash

- Static site generation (SSG) where HTML is produced once and served from a CDN.
- Cache layers that strip per-request headers.

> **Svelte's recommendation**: prefer nonce over hash when you can, because **hash will interfere with streaming SSR in the future**.

---

## 4. Algorithm Choice

By default Svelte emits SHA-256 hashes (`sha256-…`). SHA-256 is broadly supported and is the safest default.

You can also see `sha384-…` or `sha512-…` in some setups; both are stronger but the resulting hashes are longer. Pick one and stick with it across your CSP.

---

## 5. CSP and Other Inline Code

`hydratable()` is the only thing in Svelte 5 that emits inline `<script>` tags from the SSR pipeline. Inline event handlers like `<button onclick={...}>` do **not** require a nonce — they're attached at runtime as JavaScript function calls, not as parsed `<script>` content.

If you write your own inline `<script>` blocks inside a Svelte component, treat them like any other inline script: either avoid them, give them a nonce, or pre-compute their hash.

---

## 6. CSP Headers — Common Pitfalls

| Pitfall | Fix |
|---|---|
| `'unsafe-inline'` is set anywhere in `script-src` | A nonce or hash is ignored. Remove `'unsafe-inline'`. |
| `script-src` falls back to `default-src` | Be explicit: `script-src 'self' 'nonce-…'` |
| Nonce reused across responses | Generate a fresh nonce per response |
| Strict-dynamic set with a nonce | Browsers may ignore the nonce and only trust scripts loaded by the trusted script. Use `strict-dynamic` with `'nonce-…'` together; do **not** mix it with hash mode. |
| Hash mode with changing inline content | The hash won't match and the script will be blocked. Use nonce for dynamic content. |

---

## 7. What Happens if CSP Blocks the Script

If the inline script is blocked:

1. The browser parses the SSR HTML normally and shows the static markup.
2. `hydrate()` runs, **but the `hydratable()` calls inside components throw** because the bootstrap script that populated the data never executed.
3. You'll see errors like `Cannot read properties of undefined (reading 'name')` in the browser console.

Use `flushSync()` and inspect errors after hydration to catch this early.

---

## 8. Verifying CSP Compliance Locally

```bash
# Inspect CSP header
curl -I http://localhost:5173/ | grep -i content-security-policy

# Count script tags and their nonces
curl -s http://localhost:5173/ | grep -oE '<script[^>]*>' | head
```

In the browser DevTools **Console**, a blocked inline script shows:

```
Refused to execute inline script because it violates the following
Content Security Policy directive: "script-src 'self' 'nonce-abc123'".
```

---

## 9. CSP in Different Runtimes

### SvelteKit

`+page.server.js` returns `{ head, body }` and the framework lets you set headers per-route:

```js
// hooks.server.js
export async function handle({ event, resolve }) {
  const nonce = crypto.randomUUID();
  event.locals.cspNonce = nonce;

  const response = await resolve(event, {
    transformPageChunk: ({ html }) => html.replace('%csp.nonce%', nonce)
  });

  response.headers.set(
    'Content-Security-Policy',
    `script-src 'nonce-${nonce}'`
  );
  return response;
}
```

Pass the nonce through to `render()` via the `csp` option in `+page.server.js`.

### Express / Hono / Custom server

Generate a nonce in middleware, attach it to the request context, use it in `render()`, and emit it on the response header.

### Static hosting (Vercel, Netlify, S3 + CloudFront)

You cannot inject per-request nonces. Pre-render to static HTML and use **hash mode**, then set the CSP header at the CDN edge.

---

## 10. Decision Tree

```
Do you render per request?
├── Yes → use nonce mode
│         └── generate nonce, pass to render(), set CSP header
└── No (SSG) → use hash mode
              └── build → render(App, { csp: { hash: true } })
              └── set CSP header at the edge with hashes.script
```

---

## 11. Reference Summary

| Option | Purpose | Mode |
|---|---|---|
| `csp.nonce` | Mark inline scripts with a per-request nonce | Dynamic SSR |
| `csp.hash: true` | Compute SHA-256 hashes and return them in `hashes.script` | Static SSG |
| `hashes.script` | `string[]` of `'sha256-…'` values | Static SSG |
| `flushSync()` | Run pending effects to surface CSP errors in tests | Diagnostics |

See [`examples/csp-examples.md`](../examples/csp-examples.md) for concrete implementations.