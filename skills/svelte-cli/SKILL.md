---
name: svelte-cli
description: Svelte CLI（sv）技能。当用户使用 sv 创建项目（sv create）、添加集成（sv add 如 drizzle/tailwind/prettier/eslint/playwright/storybook/vitest/mdsvex/paraglide/better-auth/sveltekit-adapter/mcp/experimental）、运行类型和编译检查（sv check）、运行迁移脚本（sv migrate svelte-5/sveltekit-2/app-state/self-closing-tags/package/routes）、或开发自定义 sv add-on 时使用。
---

# Svelte CLI (`sv`)

## When to use this skill

Use this skill when the user wants to:

- Create a new SvelteKit project (`sv create`)
- Add an integration to an existing project (`sv add` with official or community add-ons)
- Run type / compile / a11y diagnostics on a Svelte project (`sv check`)
- Migrate a codebase between Svelte / SvelteKit versions (`sv migrate`)
- Build, test, and publish a custom `sv` add-on
- Read or edit `svelte.config.js` / `vite.config.js` programmatically via `sv-utils`

Do **not** use this skill for general Svelte component API questions, routing conventions, or store/state management — those belong to the `svelte` / `sveltekit` skills.

## Overview — what is `sv`?

`sv` is the official Svelte command-line toolkit for creating and maintaining Svelte(SvelteKit) applications. It ships as an npm package and is best run via `npx` / `pnpm dlx` / `bunx` / `deno run npm:sv`:

```sh
npm       -> npx sv <command>
pnpm      -> pnpm dlx sv <command>
bun       -> bunx sv <command>
deno      -> deno run npm:sv <command>
yarn      -> yarn dlx sv <command>
```

If `sv` is already a devDependency in the project, the local copy is used; otherwise it is downloaded and run ephemerally.

Five subcommands:

| Command     | Purpose                                                  |
| ----------- | -------------------------------------------------------- |
| `create`    | Scaffold a new Svelte(Kit) project + optional add-ons    |
| `add`       | Apply an add-on (official or community) to a project     |
| `check`     | Type / compile / a11y diagnostics via `svelte-check`     |
| `migrate`   | Run a codemod migration script (`svelte-migrate`)        |
| (programmatic) | `sv.create`, `sv.add`, `defineAddon`, `defineAddonOptions` |

If `npx sv` appears to do nothing, see the Quick Fixes section — this is a known npm/yarn behaviour where the local tool is preferred over downloading.

## `sv create`

```sh
npx sv create [options] [path]
```

Scaffolds a new SvelteKit project. If `path` is omitted, the current directory is used (after a non-empty check, unless `--no-dir-check` is passed).

### Options

| Flag | Description |
| ---- | ----------- |
| `--from-playground <url>` | Create a project from a [svelte.dev/playground](https://svelte.dev/playground) URL |
| `--template <name>` | `minimal`, `demo`, or `library` |
| `--types <option>` | `ts` (default to `.ts` + `lang="ts"`) or `jsdoc` |
| `--no-types` | Skip typechecking |
| `--add [add-ons...]` | Pre-pick add-ons (same syntax as `sv add`) |
| `--no-add-ons` | Skip the interactive add-ons prompt |
| `--install <pm>` | `npm` / `pnpm` / `yarn` / `bun` / `deno` |
| `--no-install` | Skip dependency installation |
| `--no-dir-check` | Allow non-empty target directory |

Common invocation:

```sh
npx sv create --template minimal --types ts --add eslint prettier my-app
```

## `sv add`

```sh
npx sv add [add-ons...]     # in a project
npx sv create --add ...     # during project creation
```

Updates an existing Svelte(SvelteKit) project with new functionality. Multiple add-ons can be space-separated. Interactive prompt runs if no add-ons are passed.

### `sv add` options

| Flag | Description |
| ---- | ----------- |
| `-C`, `--cwd <path>` | Project root |
| `--no-git-check` | Don't warn about uncommitted changes |
| `--no-download-check` | Don't warn about community add-on downloads |
| `--install <pm>` | `npm` / `pnpm` / `yarn` / `bun` / `deno` |
| `--no-install` | Don't install dependencies |

### Official add-ons

Pass them as positional args. Options use `key:value` syntax separated by `+`.

| Add-on | What it adds |
| ------ | ------------ |
| `better-auth` | Full auth setup with Drizzle adapter; email/password + optional GitHub OAuth demo pages |
| `drizzle` | ORM scaffolding for `postgresql` / `mysql` / `sqlite` with `.env`, optional Docker |
| `eslint` | ESLint flat config + `eslint-plugin-svelte`; integrates with TS and prettier |
| `experimental` | Opt into Svelte/SvelteKit experimental flags and/or `@next` line |
| `mcp` | MCP server config for AI agents (claude-code, cursor, vscode, ...) |
| `mdsvex` | Markdown preprocessor (MDX-style Svelte + Markdown) |
| `paraglide` | Inlang Paraglide i18n with Vite plugin + `reroute` / `handle` hooks |
| `playwright` | Browser test runner with config + demo test |
| `prettier` | Formatter with `.prettierrc` + integration with eslint |
| `storybook` | Storybook for SvelteKit or Svelte+Vite |
| `sveltekit-adapter` | Adapter (`auto`, `node`, `static`, `vercel`, `cloudflare`, `netlify`) |
| `tailwindcss` | Tailwind v4 Vite plugin + integration with prettier |
| `vitest` | Vite-native testing with unit/component modes |

Example with options:

```sh
npx sv add tailwindcss="plugins:typography,forms"
npx sv add drizzle="database:postgresql+client:postgres.js+docker:yes"
npx sv add sveltekit-adapter="adapter:cloudflare+cfTarget:workers"
```

### Community add-ons

> Community add-ons are **experimental**. The API may change. Svelte maintainers have **not reviewed** community add-ons for malicious code.

Community add-ons are npm packages tagged with the `sv-add` keyword. Reference by org name (looks up `@org/sv`), full package name, `file:` path, or version:

```sh
npx sv add @supacool                            # npm org lookup
npx sv add @my-org/sv@1.2.3                     # pinned version
npx sv add file:../path/to/my-addon             # local add-on
npx sv add eslint @supacool                     # mix official + community
npx sv create --add eslint @supacool my-app
```

On Windows PowerShell, escape `@` with quotes: `npx sv add '@supacool'`.

## `sv check`

```sh
npm i -D svelte-check
npx sv check
```

Runs `svelte-check`: detects unused CSS, a11y issues, and JS/TS compile errors across `.svelte`, `.svelte.ts`, `.svelte.js` files. Requires Node 16+.

### Options

| Flag | Description |
| ---- | ----------- |
| `--workspace <path>` | Workspace to scan (default: project root) |
| `--output <format>` | `human`, `human-verbose`, `machine`, `machine-verbose` |
| `--watch` | Keep alive and re-check on change |
| `--preserveWatchOutput` | Don't clear screen in watch mode |
| `--tsconfig <path>` | Use a specific `tsconfig`/`jsconfig` |
| `--no-tsconfig` | Skip `.ts`/`.js` files entirely |
| `--ignore <paths>` | Comma-separated paths to ignore (works with `--no-tsconfig`) |
| `--fail-on-warnings` | Exit non-zero on warnings |
| `--compiler-warnings <pairs>` | `code:behaviour` pairs, e.g. `css_unused_selector:ignore` |
| `--diagnostic-sources <sources>` | `js` (incl. TS), `svelte`, `css` |
| `--threshold <level>` | `warning` (default) or `error` only |

### Machine-readable output

`--output machine` and `--output machine-verbose` produce a stream of space/JSON-separated rows:

```
1590680325583 START "/path/to/workspace"
1590680326283 ERROR "codeaction.svelte" 1:16 "Cannot find module 'blubb'..."
1590680326778 WARNING "imported-file.svelte" 0:37 "Component has unused export property..."
1590680326807 COMPLETED 20 FILES 21 ERRORS 1 WARNINGS 3 FILES_WITH_PROBLEMS
1590680328921 FAILURE "Connection closed"   # only on runtime error
```

The `machine-verbose` variant emits ndjson with `start` / `end` positions, `code`, `source`, and full descriptions.

`svelte-check` does **not** support a "check only staged files" mode — it must see the whole project for cross-file type errors to be valid.

## `sv migrate`

```sh
npx sv migrate                 # interactive picker
npx sv migrate [migration]     # run a specific codemod
```

Delegates to `svelte-migrate`. Some migrations annotate your code with `// @migration` TODOs you must complete by hand.

| Migration | Purpose |
| --------- | ------- |
| `app-state` | `$app/stores` -> `$app/state` in `.svelte` files (SvelteKit 2.12+) |
| `svelte-5` | Svelte 4 -> 5: converts components to runes (`$state`, `$derived`, `$effect`, `$props`, ...) |
| `self-closing-tags` | Replaces self-closing non-void elements in `.svelte` files |
| `svelte-4` | Svelte 3 -> 4 |
| `sveltekit-2` | SvelteKit 1 -> 2 |
| `package` | `@sveltejs/package` v1 -> v2 (library authors) |
| `routes` | Pre-release SvelteKit -> SvelteKit 1 filesystem routing |

Always commit before running a migration so you can diff and revert.

## Custom add-ons (programmatic API)

Two packages form the add-on system:

- **`sv`** — where and when: workspace detection, file I/O, dependency tracking, add-on orchestration.
- **`@sveltejs/sv-utils`** — what: pure parsers, AST transforms, language helpers. No filesystem awareness.

Skeleton of a custom add-on:

```js
import { transforms } from '@sveltejs/sv-utils';
import { defineAddon, defineAddonOptions } from 'sv';

export default defineAddon({
  id: 'my-addon',
  shortDescription: 'says hello',
  options: defineAddonOptions()
    .add('who', { question: 'To whom?', type: 'string' })
    .build(),
  setup: ({ dependsOn, isKit, unsupported }) => {
    if (!isKit) unsupported('Requires SvelteKit');
    dependsOn('vitest');
  },
  run: ({ sv, options, directory }) => {
    sv.file(
      directory.kitRoutes + '/+page.svelte',
      transforms.svelte(({ ast, svelte }) => {
        svelte.addFragment(ast, `<p>Hello ${options.who}!</p>`);
      })
    );
  },
  nextSteps: ({ options }) => [`Greet ${options.who}!`]
});
```

Bootstrapping, bundling (`tsdown`), publishing (npm org required), testing (`sv/testing`'s `createSetupTest`), and `package.json` rules live in `references/custom-addon-reference.md`.

## `sv-utils` (low-level API for add-on authors)

`@sveltejs/sv-utils` is **experimental** but provides parser-aware transforms for every file type an add-on commonly edits.

### Transforms

Each transform is a curried function — invoke with a callback to get a `(content) => content` function, ready to pass to `sv.file()`.

| Transform | Callback receives | Used for |
| --------- | ----------------- | -------- |
| `transforms.script` | `{ ast, comments, content, js }` | `.js` / `.ts` files |
| `transforms.svelte` | `{ ast, content, svelte, js }` | `.svelte` components |
| `transforms.svelteScript(opts, cb)` | `{ ast, content, svelte, js }` (`ast.instance` always non-null) | Components needing guaranteed `<script>` |
| `transforms.css` | `{ ast, content, css }` | `.css` files |
| `transforms.json` | `{ data, content, json }` | `package.json`, `tsconfig.json` |
| `transforms.yaml` / `transforms.toml` | `{ data, content }` | config files |
| `transforms.text` | `{ content, text }` | `.env`, `.gitignore` |

Return `false` from the callback to abort — original content is kept. See `references/sv-utils-reference.md` for the full `js.*` / `css.*` / `svelte.*` / `json.*` / `html.*` / `text.*` helper namespaces.

### Svelte config helpers

SvelteKit config can live in `vite.config.{js,ts}` (via `sveltekit()` plugin arg) or in `svelte.config.{js,ts}`. `svelteConfig.edit` writes through both transparently — never deal with the `kit` nesting yourself:

```js
import { svelteConfig } from '@sveltejs/sv-utils';

svelteConfig.edit({ sv, cwd }, ({ ast, property, override, js }) => {
  // svelte-level option
  js.array.append(property('extensions', { fallback: js.array.create() }), '.svx');
  // kit option
  override({ adapter: js.functions.createCall({ name: 'adapter', args: [], useIdentifiers: true }) });
});
```

`svelteConfig.find(read)` returns `{ path, kind }` (`'vite'` or `'svelte'`); `svelteConfig.read(read)` returns `{ location, config, kit }` object expressions.

### Package manager helpers

`pnpm.allowBuilds('pkg-name')` returns a transform for `pnpm-workspace.yaml` that adds the package to pnpm's allow-builds config. Detects installed pnpm version:

- pnpm `>= 11` -> unified `allowBuilds: { pkg: true }` map
- pnpm `< 11` -> legacy `onlyBuiltDependencies` list

```js
if (packageManager === 'pnpm') {
  sv.file(file.findUp('pnpm-workspace.yaml'), pnpm.allowBuilds('esbuild'));
}
```

## Quick Fixes

- **`npx sv` does nothing** — npm/yarn prefer locally-installed tools. Solutions:
  - `npm exec sv ...` or `npx --yes sv ...`
  - On Windows PowerShell, `sv` collides with `Set-Variable` alias — see [sveltejs/cli#317](https://github.com/sveltejs/cli/issues/317).
  - On Linux, `sv` may collide with `runit` — see [sveltejs/cli#259](https://github.com/sveltejs/cli/issues/259).
  - Confirm: `npx --yes sv --version` should print a version.

- **`sv check` not found** — install `svelte-check` first: `npm i -D svelte-check`.

- **`sv add @community` errors on PowerShell** — quote the arg: `npx sv add '@supacool'`.

- **Migration left `@migration` TODOs** — these are manual follow-ups; search for `// @migration` and complete them.

- **`sv add` refuses because of dirty git** — pass `--no-git-check` if you're intentionally working on a dirty tree.

- **`svelte-check` fails on JS/TS files** — either provide `--tsconfig` or use `--no-tsconfig` to skip them entirely.

- **`sv add file:../path` not picking up changes** — the `demo-add` script rebuilds automatically; otherwise re-bundle your add-on before re-running.

## Gotchas

- Community add-on API is **experimental and may break**. Pin a `sv` version in `peerDependencies` (e.g. `"sv": "^0.13.0"`).
- Community add-on packages must be published under an **npm org** (`@org/...`) — plain names like `my-lib` are rejected.
- The `sv` scope on the package name resolves implicitly: `npx sv add @my-org` == `npx sv add @my-org/sv`.
- For bundled add-ons, `tsdown` is the standard. The CLI looks for `./sv` first, then `.` — so packages exporting other functionality alongside add-ons should expose the add-on via the `./sv` subpath.
- `sv check` cannot run on a subset of files (e.g. staged only). It needs the whole project for cross-file type errors to be valid.
- `--ignore` only takes effect for the file watcher when paired with `--no-tsconfig`; with `--tsconfig`, ignore is determined by the config's `exclude`.
- `svelteConfig.edit` writes through `sv.file`, so the edit shows up in the same diff as every other change — don't bypass it by reading and writing the file manually.
- `transforms.yaml` / `transforms.toml` mutate `data` in place — return value is ignored.
- pnpm `< 11` and `>= 11` use different schemas for "allow builds" — the helper handles both, but if you write directly, mirror the version logic.

## FAQ

**Q: How do I run `sv` with each package manager?**

| PM     | Command            |
| ------ | ------------------ |
| npm    | `npx sv create`    |
| pnpm   | `pnpm dlx sv create` |
| Bun    | `bunx sv create`   |
| Deno   | `deno run npm:sv create` |
| Yarn   | `yarn dlx sv create` |

**Q: Can I run `sv` from inside a project that already has it installed?**

Yes. `npx sv ...` will use the locally installed copy if present; otherwise it downloads the latest version ephemerally. This is why `sv create` works without any install step.

**Q: What's the difference between `machine` and `machine-verbose` for `sv check --output`?**

Both emit `START` / `ERROR` / `WARNING` / `COMPLETED` / `FAILURE` rows. `machine` columns are: timestamp, type, filename, line:col, message. `machine-verbose` adds end positions, diagnostic code, and source — each row as ndjson.

**Q: Are community add-ons safe to install?**

They are not reviewed by Svelte maintainers. Install at your own risk. `--no-download-check` suppresses the warning but does not add any safety.

**Q: Can I check only staged files with `sv check`?**

No. `svelte-check` requires the whole project graph. Checking a subset would miss renamed prop usages in unchanged files.

**Q: Where should `svelte.config.js` live?**

Either inside `vite.config.{js,ts}` (passed to the `sveltekit()` plugin) or as a separate `svelte.config.{js,ts}`. `sv create` keeps it in `vite.config.js`. `svelteConfig.edit` handles both transparently.

**Q: What is the minimum `sv` peer dependency for a custom add-on?**

Whatever your code uses — set it in `package.json` `peerDependencies`. Users get a compatibility warning if the installed `sv` has a different major version. Reference add-ons target `sv@^0.13.0`.

**Q: Why does `sv add` warn about uncommitted changes?**

Modifies multiple project files. The warning is a safety net; pass `--no-git-check` to bypass.