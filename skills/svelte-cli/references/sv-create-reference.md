# `sv create` reference

```
npx sv create [options] [path]
```

Scaffolds a new SvelteKit project. The target directory must be empty unless `--no-dir-check` is passed. The default directory is the current working directory.

---

## Global CLI flags

| Flag | Values | Default | Description |
| ---- | ------ | ------- | ----------- |
| `--from-playground <url>` | URL string | — | Create a project from a `svelte.dev/playground` URL. Downloads playground files, detects external dependencies, and sets up a complete SvelteKit project structure. |
| `--template <name>` | `minimal`, `demo`, `library` | `minimal` | Project template. `minimal` = barebones, `demo` = word-guessing game that works without JS, `library` = set up with `svelte-package` for publishing components. |
| `--types <option>` | `ts`, `jsdoc` | `ts` | `ts` defaults files to `.ts` and uses `lang="ts"` in components. `jsdoc` uses JSDoc syntax for type annotations. |
| `--no-types` | — | — | Skip typechecking entirely. Not recommended. |
| `--add [add-ons...]` | add-on names | — | Pre-pick add-ons to install during project creation. Same syntax as `sv add`. |
| `--no-add-ons` | — | — | Skip the interactive add-ons prompt entirely. |
| `--install <pm>` | `npm`, `pnpm`, `yarn`, `bun`, `deno` | prompt | Install dependencies with the specified package manager. |
| `--no-install` | — | — | Don't install dependencies (leaves `package.json` for the user). |
| `--no-dir-check` | — | — | Don't error if the target directory contains files. |

---

## Templates

### `minimal`
Barebones SvelteKit scaffolding:
- `src/routes/+page.svelte`
- `src/app.html`
- `src/app.d.ts`
- `vite.config.{js,ts}` with `sveltekit()` plugin
- `package.json` with `dev`, `build`, `preview` scripts
- `tsconfig.json` (if `--types ts`)

### `demo`
Everything in `minimal` plus a working word-guessing game that remains playable with JavaScript disabled — demonstrates SSR, form actions, and progressive enhancement.

### `library`
For publishing Svelte component libraries. Sets up:
- `@sveltejs/package` for building
- `pkg.exports` field in package.json
- `src/lib` with an example component
- `index.ts` entry point
- TypeScript declarations

---

## `--types`

| Value | Effect |
| ----- | ------ |
| `ts` (default) | New files are `.ts`/`.svelte` (with `lang="ts"`). Includes a `tsconfig.json`. |
| `jsdoc` | Files remain `.js`/`.svelte` but types are expressed via [JSDoc syntax](https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html). No `tsc` build step required. |
| (none) | `--no-types` skips typechecking entirely. Not recommended. |

---

## `--add` (during create)

Pass add-on names as positional args after `--add`. Same syntax as `sv add`. Multiple add-ons are space-separated.

```sh
npx sv create --add eslint prettier playwright
npx sv create --add tailwindcss="plugins:typography" vitest="usages:unit"
```

If `--add` is omitted, you can still pick add-ons interactively. `--no-add-ons` disables that prompt.

---

## `--install`

Picks a package manager for the post-scaffold install step. Equivalent to running `npm install` / `pnpm install` etc. afterwards.

If `--no-install` is passed, dependencies are not installed; the user must run the install themselves.

---

## `--no-dir-check`

By default `sv create` errors if the target directory is non-empty. This is a safety net against accidentally writing into a populated folder. `--no-dir-check` disables that check — useful for re-scaffolding or for adding files into a monorepo package.

---

## Examples

```sh
# Bare minimum
npx sv create my-app

# Skip everything interactive
npx sv create --template minimal --types ts --no-add-ons --no-install my-app

# With add-ons and install
npx sv create --template minimal --types ts --add eslint prettier --install pnpm my-app

# From playground
npx sv create --from-playground "https://svelte.dev/playground/hello-world" my-app

# Library
npx sv create --template library my-lib

# Allow non-empty dir
npx sv create --no-dir-check my-app
```

See `../examples/sv-create.md` for more.