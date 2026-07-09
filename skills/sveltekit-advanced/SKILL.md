---
name: sveltekit-advanced
description: SvelteKit 高级功能指南 - 状态管理、远程函数(Remote Functions)、环境变量、Hooks、错误处理、链接选项、Service Workers、服务端模块、快照(快照/Shallow Routing)、$app/* 模块、$lib、$service-worker。
---

# SvelteKit Advanced

Advanced SvelteKit features reference. Covers state management across server/client, the remote functions API, env vars (legacy and explicit), hooks (server/shared/universal), error handling, link options, service workers, server-only modules, snapshots, shallow routing, and the `$app/*` modules.

Source: official SvelteKit llms.txt documentation.

## When to use this skill

Use this skill when you need to:

- Choose where state lives (server context, URL, snapshot, store).
- Build type-safe client/server RPC with `query`/`form`/`command`/`prerender`.
- Read env vars safely (private vs public, static vs dynamic).
- Customize request handling via hooks (`handle`, `handleFetch`, `handleError`, `handleValidationError`, `reroute`, `transport`, `init`).
- Throw expected/unexpected errors, customize the fallback error page, type error shape via `App.Error`.
- Configure link behavior with `data-sveltekit-*` attributes.
- Add a service worker for offline / precaching.
- Prevent secret leak via `.server.*` or `$lib/server/`.
- Persist ephemeral DOM state with snapshots or route history entries with shallow routing.
- Use the `$app/*` modules (`forms`, `navigation`, `state`, `paths`, `server`, `environment`, `types`).

Do NOT use this skill for routing, form actions (legacy), load, basic setup, adapters configuration beyond env vars, or CSS/styling.

## Critical sections

### 1. State management

- **No shared server state** — module-level variables (`let user;`) leak across requests. Authenticate via cookies, persist to DB.
- **Pure `load` functions** — no side effects (no global stores). Return data instead.
- **Context for per-request state** — `setContext('user', () => data.user)` (pass a function so reactivity crosses boundaries). Reading context updated during SSR in a child does NOT propagate to the parent (it has already rendered).
- **Component state is preserved across nav** — `const` derived from `data` only computes once. Use `$derived(...)` for values that should recompute when data changes.
- **URL state** — search params for filters/sort; survives reload, affects SSR.
- **Snapshots** — disposable UI state (e.g., "is accordion open?") bound to history entry.

### 2. Remote functions (since 2.27, experimental)

Opt in via `svelte.config.js`:

```js
kit: { experimental: { remoteFunctions: true } },
compilerOptions: { experimental: { async: true } }
```

Flavors exported from `*.remote.js`:

| Flavor | Purpose | Key features |
|---|---|---|
| `query` | Read dynamic server data | Dedup, `refresh()`, `loading`/`error`/`current` |
| `query.batch` | Batch n+1 queries in one call | Returns `(input, idx) => Output` |
| `query.live` | Real-time async iterable | `connected`, `reconnect()`; first value serialized for SSR |
| `form` | Progressive-enhanced `<form>` | `<form {...createPost}>`, `createPost.fields.x.as('text')`, `validate()`, `enhance()`, `for(id)`, `preflight(schema)` |
| `command` | Imperative mutation from event handlers | Cannot be called during render |
| `prerender` | Build-time data | `inputs`, `dynamic: true` |

- Validate with any Standard Schema (Zod/Valibot).
- Args/returns serialized via devalue.
- `getRequestEvent()` works inside remote functions for cookies.
- `redirect(...)` allowed in `query`/`form`/`prerender`, NOT in `command`.
- Single-flight mutations: `getPosts().refresh()` or `getPost(id).set(...)` in server handler; `requested(getPosts, 1).refreshAll()` for client-requested refreshes (limit is DoS protection).

### 3. Environment variables

**Legacy `$env/*` (default before SvelteKit 2.63):**

| Module | Scope | Timing |
|---|---|---|
| `$env/dynamic/private` | Server only | Runtime (`process.env`-like) |
| `$env/dynamic/public` | Public (`PUBLIC_*`) | Runtime |
| `$env/static/private` | Server only | Build time (inlined) |
| `$env/static/public` | Public | Build time (inlined) |

**Explicit env vars (opt-in, default in v3):**

Enable in `svelte.config.js`: `kit.experimental.explicitEnvironmentVariables = true`. Then create `src/env.ts`:

```ts
import { defineEnvVars } from '@sveltejs/kit/hooks';
import * as v from 'valibot';
import { building } from '$app/env';

export const variables = defineEnvVars({
  API_KEY: {},                              // private
  GOOGLE_ANALYTICS_ID: { public: true },    // public
  SHOW_DEBUG_OVERLAY: { public: true, static: true },  // inlined, dead-code-eliminated
  SECRET: { schema: building ? v.optional(v.string()) : v.string() },
  CACHE_TTL_SECONDS: { description: 'How long...' }
});
```

Import from `$app/env/private` or `$app/env/public`. `$app/environment` is renamed to `$app/env`.

### 4. Hooks

Three optional files: `src/hooks.server.js`, `src/hooks.client.js`, `src/hooks.js`.

**Server (`hooks.server.js`):**

- `handle({ event, resolve })` — runs on every request; return Response or call `resolve(event, opts)`. `resolve` opts: `transformPageChunk`, `filterSerializedResponseHeaders`, `preload`. Use `sequence(...)` for multiple.
- `handleFetch({ request, fetch, event })` — rewrite cross-origin requests to internal APIs; cookie forwarding for sibling subdomains.
- `handleValidationError({ event, issues })` — customize 400 response for bad remote function args.

**Shared (`hooks.server.js` AND `hooks.client.js`):**

- `handleError({ error, event, status, message })` — for unexpected errors only. Return `{ message, ... }` -> becomes `page.error`. Must never throw. Server type: `HandleServerError`; client type: `HandleClientError`; client `event` is `NavigationEvent`.
- `init()` — runs once at startup; useful for DB connections.

**Universal (`hooks.js`):**

- `reroute({ url, fetch })` — translate URL to a different route (e.g., i18n). Pure/idempotent. Can be async since 2.18.
- `transport` — custom encoders/decoders for types crossing the server/client boundary (e.g., `Vector`).

### 5. Errors

- **Expected errors** — `error(404, 'Not found')` (or `error(404, { message, code })`) from `@sveltejs/kit`. Renders nearest `+error.svelte`, sets status code. `page.error` = the object passed.
- **Unexpected errors** — any other exception. Not exposed (generic `Internal Error`); routed through `handleError`.
- **Rendering errors** — opt in via `experimental.handleRenderingErrors`. Error passed directly to `+error.svelte` as `error` prop (not via `page.error`).
- **Responses** — custom `src/error.html` with `%sveltekit.status%` and `%sveltekit.error.message%`. Errors in root `+layout.server.js` use fallback page (root contains `+error.svelte`).
- **Type safety** — declare `App.Error` interface in `src/app.d.ts`:

```ts
declare global {
  namespace App {
    interface Error { message: string; code: string; id: string; }
  }
}
```

### 6. Link options

`data-sveltekit-*` attributes on `<a>` (or parent). Also apply to `<form method="GET">`.

| Attribute | Values | Effect |
|---|---|---|
| `preload-data` | `hover` (default), `tap` | Preload `load` data on hover/tap |
| `preload-code` | `eager`, `viewport`, `hover`, `tap` | Preload route code only |
| `reload` | boolean | Force full-page nav (also `rel="external"`) |
| `replacestate` | boolean | Replace history entry instead of push |
| `keepfocus` | boolean | Keep focus on the same element after nav |
| `noscroll` | boolean | Disable scroll-to-top after nav |

Disable in subtree with `data-sveltekit-preload-data="false"`. Respects `navigator.connection.saveData`.

### 7. Service workers

Place `src/service-worker.js` (or `src/service-worker/index.js`) — bundled and auto-registered. Disable via config to register manually.

Available from `$service-worker`: `base`, `build`, `files`, `prerendered`, `version`.

Standard pattern: cache `build + files` on install, network-first with cache fallback on fetch. Skip responses with `Cache-Control: no-store` (live queries). `build`/`prerendered` are empty arrays in dev.

### 8. Server-only modules

- `$env/static/private` and `$env/dynamic/private` — server-only.
- `$app/server` — server-only.
- Your modules — add `.server.js` suffix OR place under `$lib/server/`.

Illegal imports from browser code error at build. Use `import type` for type-only. Detection is disabled in tests (`process.env.TEST === 'true'`).

### 9. Snapshots & shallow routing

**Snapshots** — export `snapshot = { capture, restore }` from `+page.svelte` or `+layout.svelte`. Captured to `sessionStorage` before page updates; restored on history nav. Must be JSON-serializable. Don't return huge objects.

**Shallow routing** — `pushState(url, state)` / `replaceState(url, state)` create history entries without navigating. Read via `page.state`. Use `preloadData(href)` to grab `load` data, then `pushState(href, { selected: result.data })` to render another `+page.svelte` inside a modal. `page.state` is always `{}` on first SSR and during first paint.

## Quick Fixes

| Symptom | Fix |
|---|---|
| User data leaks between requests | Don't use module-level vars; use cookies + DB |
| `load` data not updating on nav | Use `$derived(...)` not `const` for values derived from `data` |
| `Cannot import $lib/server/...` | Move shared types to `import type` |
| Live query stream stops | Reconnect with `.reconnect()`; don't cache `no-store` responses in SW |
| `error(404, ...)` shows blank page | Add `+error.svelte` nearest to the route |
| Server-only env leaked to client | Use `$env/static/private` or `.server` naming |
| Validation 400 generic message | Implement `handleValidationError` |
| Component state lost on nav | Wrap with `{#key page.url.pathname}<X/>{/key}` to force remount |

## Gotchas

- **`load` must be pure** — no global store writes. Return the data instead.
- **Context updates during SSR don't propagate up** — pass state down to avoid flash on hydration.
- **`query.batch`** returns a single function that maps individual inputs to outputs; not a direct array of results.
- **`query.live` on SSR** returns ONLY the first yielded value then closes.
- **`command` cannot be called during render** — invoke from event handlers.
- **`requested` requires a `limit`** — DoS protection; pass `Infinity` only if explicitly safe.
- **`error()` no longer needs `throw`** in SvelteKit 2.x.
- **`page.state` is `{}` on SSR and first paint** — don't rely on it for critical render.
- **`reroute` must be pure/idempotent** — its result is cached per URL on the client.
- **Service worker `build`/`prerendered` empty in dev** — test in production build.
- **Cookie forwarding for sibling subdomains requires manual `handleFetch`** — SvelteKit can't tell which parent-domain cookie belongs to which subdomain.
- **`handleError` must not throw** — wrap risky work in try/catch.

## FAQ

**Q: $app/state vs $app/stores?**
A: `$app/state` (since 2.12) is rune-based and reactive. `$app/stores` is the legacy store-based version. Use `$app/state` with Svelte 5.

**Q: query vs prerender?**
A: `query` for dynamic data; `prerender` for build-time-frozen data that can be served from a CDN. `query` cannot be used on a fully prerendered page.

**Q: form vs command?**
A: `form` is progressive-enhanced — works without JS, spreads onto `<form>`. `command` is JS-only, called from event handlers. Prefer `form`.

**Q: $env/dynamic vs $env/static?**
A: `dynamic` reads at runtime (e.g., `process.env`); `static` inlined at build time (enables dead-code elimination). Use `static` for build-time-known values.

**Q: $env/static/private vs $env/dynamic/private?**
A: Same access restriction; difference is when the value is read. Static gives DCE; dynamic allows runtime override (e.g., `MY_FLAG=1 npm run dev`).

**Q: Can I use $lib/server modules from +page.svelte?**
A: No — server-only modules cannot be imported by client code, transitively. SvelteKit errors at build.

**Q: How do I keep focus on a search input after submit?**
A: Add `data-sveltekit-keepfocus` to the `<form>`.

**Q: How do I throw a 404 from a load?**
A: `import { error } from '@sveltejs/kit'; error(404, 'Not found');` (don't `throw` in SvelteKit 2).

**Q: Should I use Sentry?**
A: Yes — initialize in `handleError` of both `hooks.server.js` and `hooks.client.js`. Server uses `HandleServerError`; client uses `HandleClientError` with `NavigationEvent`.

## Examples

| Topic | File |
|---|---|
| State management with context + URL + derived | `examples/state-management.md` |
| Remote query/form/command/batch/live | `examples/remote-functions.md` |
| Env vars (legacy + explicit) | `examples/env-vars.md` |
| Server + universal hooks | `examples/hooks.md` |
| Expected/unexpected errors + App.Error | `examples/errors.md` |
| data-sveltekit-* link options | `examples/link-options.md` |
| Service worker precache + offline | `examples/service-worker.md` |
| $app/* modules (forms, navigation, state, paths, server) | `examples/app-modules.md` |
| Snapshots + shallow routing | `examples/shallow-snapshots.md` |

## References

| Topic | File |
|---|---|
| State management | `references/state-management.md` |
| Remote functions | `references/remote-functions.md` |
| Environment variables | `references/env-vars.md` |
| Hooks | `references/hooks.md` |
| Errors | `references/errors.md` |
| Link options | `references/link-options.md` |
| Service workers | `references/service-worker.md` |
| Server-only modules | `references/server-only.md` |
| $app/* modules | `references/app-modules.md` |
| Snapshots & shallow routing | `references/shallow-snapshots.md` |