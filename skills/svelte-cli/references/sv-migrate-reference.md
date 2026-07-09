# `sv migrate` reference

```
npx sv migrate            # interactive picker
npx sv migrate <name>     # run a specific migration
```

`sv migrate` delegates to the [`svelte-migrate`](https://www.npmjs.com/package/svelte-migrate) package. Migrations are codemods — always commit your work before running so you can diff / revert.

Some migrations annotate the codebase with `// @migration` TODOs that must be completed by hand. Search the codebase for `// @migration` after running.

---

## Available migrations

| Migration | From | To | What it does |
| --------- | ---- | -- | ------------ |
| `app-state` | SvelteKit 2.0–2.11 | SvelteKit 2.12+ | `$app/stores` (`$page`, `$navigating`) -> `$app/state` reactive exports in `.svelte` files. |
| `svelte-5` | Svelte 4 | Svelte 5 | Converts components to runes (`$state`, `$derived`, `$effect`, `$props`, ...), updates event dispatcher usage, slot -> snippet. |
| `self-closing-tags` | (any) | (any) | Replaces self-closing non-void elements in `.svelte` files (e.g. `<div />` -> `<div></div>`). |
| `svelte-4` | Svelte 3 | Svelte 4 | Removes deprecated APIs: slot props, `<svelte:fragment>` semantics, etc. |
| `sveltekit-2` | SvelteKit 1 | SvelteKit 2 | Updates removed APIs and renamed config keys. |
| `package` | `@sveltejs/package` v1 | v2 | Library authors: updates build config for `svelte-package` v2 output layout. |
| `routes` | Pre-release SvelteKit (`routes/` convention) | SvelteKit 1 | Moves routes from `routes/` directory to `src/routes/`. |

---

## `app-state`

```sh
npx sv migrate app-state
```

The SvelteKit 2.12 release replaced `$app/stores` with `$app/state`. The new exports are not stores — they're plain reactive objects meant to be used with `$derived` (in Svelte 5) or `$:` (in Svelte 4).

Before:

```svelte
<script>
  import { page } from '$app/stores';
</script>
<h1>{$page.url.pathname}</h1>
```

After (in Svelte 5):

```svelte
<script>
  import { page } from '$app/state';
</script>
<h1>{page.url.pathname}</h1>
```

See the [migration guide](https://svelte.dev/docs/kit/migrating-to-sveltekit-2#SvelteKit-2.12:-$app-stores-deprecated).

---

## `svelte-5`

```sh
npx sv migrate svelte-5
```

The largest behavioural migration. Converts:

| Svelte 4 | Svelte 5 |
| -------- | -------- |
| `let count = 0;` (reactive) | `let count = $state(0);` |
| `$: doubled = count * 2;` | `let doubled = $derived(count * 2);` |
| `$: console.log(count);` | `$effect(() => console.log(count));` |
| `export let name;` | `let { name } = $props();` |
| `<slot />` | `{@render children()}` (with `let { children } = $props();`) |
| `createEventDispatcher` | Callback props |
| `bind:this` on components | Still works; component refs return export bindings |
| `writable` / `readable` stores | `$state` in runes mode |

This codemod also updates `package.json` to use `svelte@^5` and rewrites `vite.config` to use the runes-compatible Vite plugin if needed.

See the [v5 migration guide](https://svelte.dev/docs/svelte/v5-migration-guide).

---

## `self-closing-tags`

```sh
npx sv migrate self-closing-tags
```

The HTML spec disallows self-closing non-void elements (e.g. `<div />`). Some Svelte 4 setups accepted them; this codemod rewrites them to the explicit form.

Before:
```svelte
<div />
```

After:
```svelte
<div></div>
```

See sveltejs/kit#12128.

---

## `svelte-4`

```sh
npx sv migrate svelte-4
```

A smaller jump than `svelte-5`. Main changes:
- `slot="..."` -> `let:` style slot props where applicable
- Removed `<svelte:fragment>` for some edge cases
- New `transition:` directive syntax
- Some lifecycle hook timing changes

See the [v4 migration guide](https://svelte.dev/docs/svelte/v4-migration-guide).

---

## `sveltekit-2`

```sh
npx sv migrate sveltekit-2
```

Updates for SvelteKit 2 breaking changes:
- `+page.svelte` / `+layout.svelte` data flow changes
- Removed `getSession` / `getStores` etc.
- Updated `error` / `redirect` signatures
- Renamed config keys

See the [SvelteKit 2 migration guide](https://svelte.dev/docs/kit/migrating-to-sveltekit-2).

---

## `package`

```sh
npx sv migrate package
```

For **library authors** using `@sveltejs/package`. v2 of `svelte-package` reorganised the output layout and config. See sveltejs/kit#8922 for the full list.

---

## `routes`

```sh
npx sv migrate routes
```

For projects on the pre-release SvelteKit filesystem routing convention (which placed routes in a top-level `routes/` directory) — moves them to the canonical `src/routes/`. See sveltejs/kit#5774.

---

## Workflow tips

- **Always commit before running.** Migrations are codemods; mistakes are recoverable via git.
- **Run migrations sequentially when chaining.** Don't try to skip a major version.
- **Search for `@migration` afterwards.** Any remaining items are manual.
- **Pair with `sv check`.** After migrating, run `npx sv check` to surface remaining type errors.

```sh
git commit -am "pre-migration"
npx sv migrate svelte-5
git add -A && git commit -am "svelte 5 migration"
npx sv check
rg "@migration" -t svelte   # manual follow-ups
```