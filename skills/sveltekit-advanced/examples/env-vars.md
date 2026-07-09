# Environment Variables Examples

Reading env vars in SvelteKit — legacy `$env/*` and the new explicit `$app/env/*`.

## Legacy `$env/*` modules (default before SvelteKit 2.63)

### 1. .env file

```env
# .env.local
API_KEY=19f401ba-e8b0-48c4-8c77-b0ebb26d97fe
PUBLIC_BASE_URL=https://example.com
```

In dev, variables from `.env` and `.env.local` are loaded by Vite.

### 2. Private dynamic — server only

```js
// +page.server.js or hooks.server.js
import { env } from '$env/dynamic/private';

export async function load() {
  return { apiKey: env.API_KEY };
}
```

Cannot be imported from client code.

### 3. Public dynamic — only PUBLIC_*

```js
// +page.svelte (client OK)
import { env } from '$env/dynamic/public';

console.log(env.PUBLIC_BASE_URL);  // 'https://example.com'
console.log(env.API_KEY);          // undefined (not PUBLIC_)
```

### 4. Private static — build-time inlined

```js
// Server only
import { API_KEY } from '$env/static/private';

console.log(API_KEY);  // value frozen at build time
```

Enables dead-code elimination.

### 5. Public static — client-safe build-time

```js
// Client-safe
import { PUBLIC_BASE_URL } from '$env/static/public';

console.log(PUBLIC_BASE_URL);
```

### 6. Override at dev time

```bash
MY_FEATURE_FLAG=enabled npm run dev
```

For dynamic vars this changes at runtime; for static vars it would require rebuild.

### 7. Declare undeclared vars for types

```env
# .env (even empty)
MY_FEATURE_FLAG=
```

Ensures TS doesn't complain before deployment sets it.

## Explicit env vars (SvelteKit 2.63+ opt-in, default in v3)

### 8. Enable explicit env vars

```js
// svelte.config.js
export default {
  kit: {
    experimental: {
      explicitEnvironmentVariables: true
    }
  }
};
```

### 9. Define env vars in src/env.ts

```ts
// src/env.ts
import { defineEnvVars } from '@sveltejs/kit/hooks';
import * as v from 'valibot';
import { building } from '$app/env';

export const variables = defineEnvVars({
  // Private — server only
  API_KEY: {},

  // Public — available in browser
  GOOGLE_ANALYTICS_ID: {
    public: true,
    schema: v.pipe(v.string(), v.regex(/G-[A-Z0-9]+/))
  },

  // Inlined — DCE-friendly
  SHOW_DEBUG_OVERLAY: {
    public: true,
    static: true,
    schema: v.pipe(
      v.optional(v.string(), ''),
      v.transform((s) => s !== '')
    )
  },

  // Required at runtime, optional during build
  SECRET: {
    schema: building ? v.optional(v.string()) : v.string()
  },

  // Documented
  CACHE_TTL_SECONDS: {
    description: 'How long to cache responses, in seconds'
  }
});
```

### 10. Import explicit env vars

```js
// Server only
import { API_KEY } from '$app/env/private';

// Client-safe
import { GOOGLE_ANALYTICS_ID } from '$app/env/public';
```

### 11. Public var in app.html

```html
<!-- src/app.html -->
<script
  async
  src="https://www.googletagmanager.com/gtag/js?id=%sveltekit.env.GOOGLE_ANALYTICS_ID%"
></script>
<script>
  window.dataLayer ??= [];
  function gtag() { dataLayer.push(arguments); }
  gtag('js', new Date());
  gtag('config', '%sveltekit.env.GOOGLE_ANALYTICS_ID%');
</script>
```

### 12. Static variable enables DCE

```svelte
<script>
  import { SHOW_DEBUG_OVERLAY } from '$app/env/public';
  import DebugOverlay from '$lib/components/DebugOverlay.svelte';
</script>

{#if SHOW_DEBUG_OVERLAY}
  <DebugOverlay />
{/if}
```

When `SHOW_DEBUG_OVERLAY=true` is set at build, the component is included; otherwise DCE removes it.

### 13. Building vs running check

```js
import { building } from '$app/env';
import { initialiseDatabase } from '$lib/server/database';

if (!building) {
  initialiseDatabase();
}

export function load() {
  // ...
}
```

`building` is true during prerender and `vite build`.

### 14. Production env loading (adapter-node)

Production `.env` files are NOT auto-loaded. Either:

```bash
node -r dotenv/config build
```

or (Node 20.6+):

```bash
node --env-file=.env build
```

### 15. Deployment config env vars (adapter-node)

```bash
HOST=127.0.0.1 PORT=4000 ORIGIN=https://my.site node build
```

For proxies:

```bash
PROTOCOL_HEADER=x-forwarded-proto \
HOST_HEADER=x-forwarded-host \
ADDRESS_HEADER=True-Client-IP \
XFF_DEPTH=3 \
node build
```

### 16. Validation failures stop startup

```ts
// If GOOGLE_ANALYTICS_ID is "wrong-id", it fails the regex above
// and the app refuses to start (or build).
```

Use `building ? v.optional(...) : v.string()` to allow missing values during build but require at runtime.

### 17. Transform a value

```ts
PORT: {
  description: 'HTTP port',
  schema: v.pipe(
    v.optional(v.string(), '3000'),
    v.transform(Number)
  )
}
```

Then `import { PORT } from '$app/env/private'; PORT === 3000` (number).