# adapter-netlify

Deploy to Netlify Functions (Node) or Edge Functions (Deno).

## 1. Install

```sh
npm i -D @sveltejs/adapter-netlify
```

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-netlify';
export default { kit: { adapter: adapter() } };
```

## 2. netlify.toml

```toml
# netlify.toml
[build]
  command = "npm run build"
  publish = "build"
```

`publish` must match where the adapter writes output.

## 3. Edge Functions (Deno)

```js
adapter({ edge: true })
```

Lower latency (closer to user), but no Node APIs (`fs`, etc).

## 4. Split functions

By default one function handles everything. Split per route for faster cold starts:

```js
adapter({ split: true })
```

Or per route via `export const config`:

```js
// src/routes/api/heavy/+server.js
/** @type {import('@sveltejs/adapter-netlify').Config} */
export const config = { split: true };
```

Note: `split: true` and `edge: true` are mutually exclusive.

## 5. Environment variables

Set in Netlify UI or `netlify.toml`:

```toml
[build.environment]
  MY_SECRET = "value"
```

```js
import { MY_SECRET } from '$env/static/private';
```

## 6. Access Netlify Identity / Functions context

```js
// src/routes/profile/+page.server.js
export const load = (event) => {
  const context = event.platform.context; // Netlify context
  const user = context.clientContext?.user;
  return { user };
};
```

`context` is `undefined` on non-Netlify adapters — guard accordingly.

## 7. Netlify Forms

Add a hidden `form-name` input and prerender the page:

```svelte
<!-- src/routes/contact/+page.svelte -->
<form method="POST" netlify>
  <input type="hidden" name="form-name" value="contact" />
  <input name="email" type="email" />
  <textarea name="message"></textarea>
  <button>Send</button>
</form>
```

```js
// src/routes/contact/+page.js
export const prerender = true;
```

## 8. Custom Netlify Functions alongside SvelteKit

```toml
# netlify.toml
[build]
  command = "npm run build"
  publish = "build"
[functions]
  directory = "functions"
```

```js
// functions/hello.js
export default async () => new Response('Hello');
```

These are independent of SvelteKit. Reach them at `/.netlify/functions/hello`.

## 9. Local dev with Netlify CLI

```sh
npm i -D netlify-cli
netlify dev
```

Emulates Netlify Functions + redirects + headers locally.

## 10. Skew protection

Netlify has a similar feature to Vercel — enable in Site Settings → Edge Functions → Skew Protection. SvelteKit's built-in skew detection (auto-reload on error) is enabled by default.

## See also

- [Adapter comparison](../references/adapters-comparison.md)
