# `sv add` examples — every official add-on

Run from inside your project root (or pass `-C <path>`).

Option syntax: `add-on="key:value+key:value"`.

---

## 1. `drizzle` — ORM with database + client + Docker

```sh
# Interactive picker
npx sv add drizzle

# PostgreSQL with postgres.js client + Docker Compose
npx sv add drizzle="database:postgresql+client:postgres.js+docker:yes"

# MySQL with planetscale
npx sv add drizzle="database:mysql+client:planetscale"

# SQLite with better-sqlite3
npx sv add drizzle="database:sqlite+client:better-sqlite3"

# SQLite + libsql (Turso-compatible)
npx sv add drizzle="database:sqlite+client:libsql"
```

Creates `.env`, a server-only DB module, and is compatible with `better-auth`.

## 2. `tailwindcss` — utility CSS

```sh
npx sv add tailwindcss

# With typography + forms plugins
npx sv add tailwindcss="plugins:typography,forms"

# Typography only
npx sv add tailwindcss="plugins:typography"
```

Installs the Tailwind v4 Vite plugin, updates `app.css` / `layout.css`, and integrates with `prettier` if present.

## 3. `prettier` — formatter

```sh
npx sv add prettier
```

Adds `.prettierrc`, `.prettierignore`, format scripts, and (if `eslint` is installed) wires ESLint to defer to Prettier.

## 4. `eslint` — linter

```sh
npx sv add eslint
```

Adds flat `eslint.config.js` with `eslint-plugin-svelte`. Integrates with TypeScript and Prettier automatically.

## 5. `mdsvex` — Markdown preprocessor

```sh
npx sv add mdsvex
```

Lets you use Svelte components in Markdown and vice versa. Configures `svelte.config.js` with the `mdsvex` preprocessor.

## 6. `vitest` — unit/component testing

```sh
# Both unit and component
npx sv add vitest="usages:unit,component"

# Just unit tests
npx sv add vitest="usages:unit"

# Just component tests
npx sv add vitest="usages:component"
```

Adds `@vitest/browser`-aware setup for Svelte components and demo test files.

## 7. `playwright` — browser E2E

```sh
npx sv add playwright
```

Adds Playwright config, scripts, `.gitignore` updates, and a demo test.

## 8. `storybook` — component workshop

```sh
npx sv add storybook
```

Runs `npx storybook init` under the hood with defaults appropriate for SvelteKit or Svelte+Vite projects (auto-detected).

## 9. `paraglide` — i18n

```sh
# English + Spanish
npx sv add paraglide="languageTags:en,es"

# With demo page
npx sv add paraglide="languageTags:en,es+demo:yes"

# No demo
npx sv add paraglide="languageTags:en,fr,de"
```

Sets up Inlang project, Paraglide Vite plugin, `reroute` + `handle` hooks, `lang`/`dir` on `<html>`.

## 10. `mcp` — Svelte MCP for AI agents

```sh
# Pick all supported IDEs
npx sv add mcp="ide:claude-code,cursor,vscode,gemini,opencode"

# Single IDE
npx sv add mcp="ide:cursor"

# Local vs remote setup
npx sv add mcp="setup:local"
npx sv add mcp="setup:remote"
```

Writes MCP config and an `AGENTS.md` for AI-assisted development.

## 11. `better-auth` — auth

```sh
# Email + password demo only (default)
npx sv add better-auth="demo:password"

# GitHub OAuth only
npx sv add better-auth="demo:github"

# Both
npx sv add better-auth="demo:password,github"
```

Requires `drizzle` (will be added automatically if missing).

## 12. `sveltekit-adapter` — deployment target

```sh
# Standalone Node server
npx sv add sveltekit-adapter="adapter:node"

# Static site
npx sv add sveltekit-adapter="adapter:static"

# Vercel
npx sv add sveltekit-adapter="adapter:vercel"

# Cloudflare Workers
npx sv add sveltekit-adapter="adapter:cloudflare+cfTarget:workers"

# Cloudflare Pages
npx sv add sveltekit-adapter="adapter:cloudflare+cfTarget:pages"

# Netlify
npx sv add sveltekit-adapter="adapter:netlify"

# Auto-detect platform
npx sv add sveltekit-adapter="adapter:auto"
```

## 13. `experimental` — opt into upcoming features

```sh
# Bump kit + adapter to the @next line
npx sv add experimental="versions:kit"

# Enable async components + remote functions
npx sv add experimental="features:async,remoteFunctions"

# Multiple flags
npx sv add experimental="features:async,remoteFunctions,explicitEnvironmentVariables,handleRenderingErrors,forkPreloads"
```

Available flags: `async`, `remoteFunctions`, `explicitEnvironmentVariables`, `handleRenderingErrors`, `forkPreloads`.

---

## Combining add-ons

Multiple add-ons in one call are space-separated; options for each come after `=`:

```sh
npx sv add eslint prettier playwright vitest
npx sv add tailwindcss="plugins:typography" prettier eslint
npx sv add drizzle="database:postgresql+client:postgres.js" better-auth="demo:password"
```

## Adding to a different cwd

```sh
npx sv add -C ../other-project prettier
```

## Skipping install / git check

```sh
npx sv add --no-install eslint
npx sv add --no-git-check --no-install prettier
```

## Community add-ons alongside official ones

```sh
npx sv add eslint @supacool
npx sv add @my-org/sv@1.2.3
npx sv add file:../local-addon-dev
```

On Windows PowerShell, quote `@` args: `npx sv add '@supacool'`.