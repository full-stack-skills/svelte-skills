# Official add-ons reference

All 13 official add-ons. Use via `npx sv add <name>` or `npx sv create --add <name>`.

Option syntax: `add-on="key:value+key:value"`. Multiple options are `+`-separated.

---

## `better-auth`

[Better Auth](https://www.better-auth.com/) — framework-agnostic TypeScript auth library, wired for SvelteKit with Drizzle.

**Provides:**
- Full auth setup for SvelteKit with Drizzle as the database adapter
- Email/password authentication enabled by default
- Optional demo registration and login pages

**Requires:** Drizzle (auto-installed if missing).

### Options

| Option | Values | Default | Description |
| ------ | ------ | ------- | ----------- |
| `demo` | `password`, `github` | `password` | Which demo pages to include. Comma-separated to enable both. |

```sh
npx sv add better-auth="demo:password"
npx sv add better-auth="demo:github"
npx sv add better-auth="demo:password,github"
```

---

## `drizzle`

[Drizzle ORM](https://orm.drizzle.team/) — TypeScript ORM with relational and SQL-like query APIs.

**Provides:**
- Database access kept in SvelteKit server files
- `.env` for credentials
- Compatibility with the `better-auth` add-on
- Optional Docker Compose configuration

### Options

| Option | Values | Description |
| ------ | ------ | ----------- |
| `database` | `postgresql`, `mysql`, `sqlite` | Required. Which database backend. |
| `client` | (depends on `database`) | SQL client. See below. |
| `docker` | `yes`, `no` | Add Docker Compose. Only available for `postgresql` / `mysql`. |

**Clients per database:**

| Database | Available clients |
| -------- | ----------------- |
| `postgresql` | `postgres.js`, `neon` |
| `mysql` | `mysql2`, `planetscale` |
| `sqlite` | `better-sqlite3`, `libsql`, `turso` |

```sh
npx sv add drizzle
npx sv add drizzle="database:postgresql+client:postgres.js"
npx sv add drizzle="database:postgresql+client:postgres.js+docker:yes"
```

Drizzle supports many more drivers than those listed; pick one as a placeholder and swap after.

---

## `eslint`

[ESLint](https://eslint.org/) — finds and fixes code problems.

**Provides:**
- Required packages installed including `eslint-plugin-svelte`
- An `eslint.config.js` (flat config)
- Updated `.vscode/extensions.json`
- Configured to work with TypeScript and Prettier if those packages are present

```sh
npx sv add eslint
```

(No options.)

---

## `experimental`

Enables Svelte and SvelteKit experimental features, and optionally moves packages to their `next` line.

### Options

| Option | Values | Description |
| ------ | ------ | ----------- |
| `versions` | `kit` | Bump `@sveltejs/kit` (and adapter + required peers) to the `next` line. |
| `features` | (comma-separated list) | Experimental flags to enable. |

**Available flags:**

| Flag | Description |
| ---- | ----------- |
| `async` | `await` in components |
| `remoteFunctions` | Remote functions |
| `explicitEnvironmentVariables` | Explicit env vars (SvelteKit `^2` only) |
| `handleRenderingErrors` | Rendering error boundaries |
| `forkPreloads` | Forked preloading |

```sh
npx sv add experimental="versions:kit"
npx sv add experimental="features:async,remoteFunctions"
npx sv add experimental="versions:kit+features:async"
```

---

## `mcp`

[Svelte MCP](https://svelte.dev/docs/ai/overview) — helps LLMs write better Svelte code.

**Provides:**
- An MCP configuration for [local](https://svelte.dev/docs/ai/local-setup) or [remote](https://svelte.dev/docs/ai/remote-setup) setup
- An `AGENTS.md` for agent guidance

### Options

| Option | Values | Description |
| ------ | ------ | ----------- |
| `ide` | `claude-code`, `cursor`, `gemini`, `opencode`, `vscode`, `other` | Which IDE to configure (comma-separated for multiple). |
| `setup` | `local`, `remote` | Local vs remote MCP setup. |

```sh
npx sv add mcp="ide:cursor"
npx sv add mcp="ide:claude-code,cursor,vscode"
npx sv add mcp="setup:local"
npx sv add mcp="setup:remote"
```

---

## `mdsvex`

[mdsvex](https://mdsvex.pngwn.io) — markdown preprocessor for Svelte components (MDX for Svelte).

**Provides:**
- `mdsvex` installed
- Configured in `svelte.config.js` as a preprocessor

```sh
npx sv add mdsvex
```

(No options.)

---

## `paraglide`

[Inlang Paraglide](https://inlang.com/m/gerre34r/library-inlang-paraglideJs) — compiler-based i18n with tree-shakable messages.

**Provides:**
- Inlang project settings
- Paraglide Vite plugin
- SvelteKit `reroute` and `handle` hooks
- `text-direction` and `lang` attributes on `<html>`
- Updated `.gitignore`
- Optional demo page

### Options

| Option | Values | Description |
| ------ | ------ | ----------- |
| `languageTags` | IETF BCP 47 tags (comma-separated) | Languages to support, e.g. `en,es,fr`. |
| `demo` | `yes`, `no` | Whether to generate a demo page. |

```sh
npx sv add paraglide="languageTags:en,es"
npx sv add paraglide="languageTags:en,fr,de+demo:yes"
```

---

## `playwright`

[Playwright](https://playwright.dev) — browser testing.

**Provides:**
- Scripts in `package.json` (`test`, `test:unit`, `test:e2e`)
- `playwright.config.{js,ts}`
- Updated `.gitignore`
- A demo test

```sh
npx sv add playwright
```

(No options.)

---

## `prettier`

[Prettier](https://prettier.io) — opinionated code formatter.

**Provides:**
- Format scripts in `package.json`
- `.prettierignore` and `.prettierrc`
- Updates to ESLint config (if `eslint` is present) to defer to Prettier

```sh
npx sv add prettier
```

(No options.)

---

## `storybook`

[Storybook](https://storybook.js.org/) — frontend component workshop.

**Provides:**
- Runs `npx storybook init` with defaults for SvelteKit or Svelte+Vite (auto-detected)
- Mocking for many SvelteKit modules
- Automatic link handling

```sh
npx sv add storybook
```

(No options.)

---

## `sveltekit-adapter`

SvelteKit adapter for deployment to a specific platform.

### Options

| Option | Values | Description |
| ------ | ------ | ----------- |
| `adapter` | `auto`, `node`, `static`, `vercel`, `cloudflare`, `netlify` | Required. Which adapter to install. |
| `cfTarget` | `workers`, `pages` | Only for `cloudflare` adapter. Where to deploy. |

| Adapter | Package | Notes |
| ------- | ------- | ----- |
| `auto` | `@sveltejs/adapter-auto` | Detects platform, less configurable. |
| `node` | `@sveltejs/adapter-node` | Standalone Node server. |
| `static` | `@sveltejs/adapter-static` | SSG (static site generation). |
| `vercel` | `@sveltejs/adapter-vercel` | Vercel. |
| `cloudflare` | `@sveltejs/adapter-cloudflare` | Cloudflare (use `cfTarget`). |
| `netlify` | `@sveltejs/adapter-netlify` | Netlify. |

```sh
npx sv add sveltekit-adapter="adapter:node"
npx sv add sveltekit-adapter="adapter:cloudflare+cfTarget:workers"
npx sv add sveltekit-adapter="adapter:static"
```

---

## `tailwindcss`

[Tailwind CSS](https://tailwindcss.com/) — utility CSS.

**Provides:**
- Tailwind v4 setup following the Tailwind for SvelteKit guide
- Tailwind Vite plugin
- Updated `app.css` / `layout.css` (SvelteKit) or `app.css` + `App.svelte` (non-SvelteKit Vite)
- Integration with `prettier` if installed

### Options

| Option | Values | Description |
| ------ | ------ | ----------- |
| `plugins` | `typography`, `forms` | Comma-separated. |

```sh
npx sv add tailwindcss
npx sv add tailwindcss="plugins:typography"
npx sv add tailwindcss="plugins:typography,forms"
```

---

## `vitest`

[Vitest](https://vitest.dev/) — Vite-native testing framework.

**Provides:**
- Required packages installed
- Test scripts in `package.json`
- Client/server-aware Svelte testing setup in Vite config
- Demo tests

### Options

| Option | Values | Description |
| ------ | ------ | ----------- |
| `usages` | `unit`, `component` | Comma-separated. Which test types to enable. |

```sh
npx sv add vitest
npx sv add vitest="usages:unit"
npx sv add vitest="usages:unit,component"
npx sv add vitest="usages:component"
```