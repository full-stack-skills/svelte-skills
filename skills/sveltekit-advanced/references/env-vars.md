# Environment Variables Reference

## Module summary (SvelteKit 2.63 and earlier default)

| Module | Scope | Read at | Variables included |
|--------|-------|---------|--------------------|
| `$env/dynamic/private` | Server only | Runtime | All non-PUBLIC_* |
| `$env/dynamic/public` | Client + server | Runtime | Only PUBLIC_* (default prefix) |
| `$env/static/private` | Server only | Build time | All non-PUBLIC_* (inlined) |
| `$env/static/public` | Client + server | Build time | Only PUBLIC_* (inlined) |

Static = inlined → enables dead-code elimination.
Dynamic = read at runtime → can override with `VAR=val npm run dev`.

## Prefixes (configurable)

- `config.kit.env.publicPrefix` — default `PUBLIC_`.
- `config.kit.env.privatePrefix` — default empty.

## Explicit env vars (SvelteKit 2.63+ opt-in, default in v3)

Enable: `kit.experimental.explicitEnvironmentVariables = true`.

Create `src/env.ts`:

```ts
import { defineEnvVars } from '@sveltejs/kit/hooks';
import * as v from 'valibot';
import { building } from '$app/env';

export const variables = defineEnvVars({
  API_KEY: {},                          // private, no validation
  GA_ID: { public: true },              // public
  SHOW_DEBUG: {                         // public, inlined, with schema
    public: true,
    static: true,
    schema: v.pipe(v.optional(v.string(), ''), v.transform(s => s !== ''))
  },
  SECRET: { schema: building ? v.optional(v.string()) : v.string() },
  TTL: { description: 'cache TTL' }
});
```

Import:

```js
import { API_KEY } from '$app/env/private';
import { GA_ID } from '$app/env/public';
```

`$app/env` (alias for `$app/environment`) is unchanged.

## `App.html` substitutions

Use `%sveltekit.env.VAR_NAME%` in `src/app.html`. Only available for explicit-env vars marked `public: true`.

## Building vs runtime check

```js
import { building } from '$app/environment';

if (!building) {
  // code that should NOT run during vite build / prerender
}
```

`building` is `true` during prerendering too.

## .env file loading

- `dev` / `preview`: Vite loads `.env`, `.env.local`, `.env.[mode]`.
- `prod`: depends on adapter. For `adapter-node`, install `dotenv` and `node -r dotenv/config build` (or Node 20.6+: `node --env-file=.env build`).

## Type declarations

Declare unused vars for types:

```env
MY_FLAG=
```

This lets TypeScript see the type before deployment sets the value.

## Validation

- Standard Schema (Zod/Valibot) via `schema`.
- `building ? v.optional(...) : v.string()` for build-time-optional, runtime-required.
- Transforms supported (string → number, JSON parse, etc.).

## Static vs dynamic rule of thumb

- Use `static` for build-time-known values that affect bundling (feature flags that remove code).
- Use `dynamic` for runtime-overridable values (DB URLs, secrets injected at deploy).

## adapter-node env vars (deployment)

| Var | Default | Purpose |
|-----|---------|---------|
| `HOST` | `0.0.0.0` | Bind address |
| `PORT` | `3000` | Bind port |
| `SOCKET_PATH` | — | Unix socket path (overrides HOST/PORT) |
| `ORIGIN` | — | Public origin URL |
| `PROTOCOL_HEADER` | — | Reverse proxy protocol header |
| `HOST_HEADER` | — | Reverse proxy host header |
| `PORT_HEADER` | — | Reverse proxy port header |
| `ADDRESS_HEADER` | — | Client IP header |
| `XFF_DEPTH` | — | Trusted proxy count for X-Forwarded-For |
| `BODY_SIZE_LIMIT` | `512kb` | Max body size |
| `SHUTDOWN_TIMEOUT` | `30` | Graceful shutdown seconds |
| `IDLE_TIMEOUT` | — | Systemd idle sleep |
| `envPrefix` (config) | `''` | Custom prefix for all of the above |