# `sv migrate` examples

`sv migrate` runs codemods from the `svelte-migrate` package. Always **commit before running** so you can diff / revert.

Some migrations leave `// @migration` comments for manual follow-ups — search the codebase for `// @migration` after running.

---

## 1. Interactive migration picker

```sh
npx sv migrate
```

Lists all available migrations; pick one in the terminal UI.

## 2. Migrate Svelte 4 -> Svelte 5 (runes)

```sh
npx sv migrate svelte-5
```

Converts:
- `let` to `$state`
- `$:` to `$derived` / `$effect`
- `export let` to `$props`
- `<slot>` to `{@render children()}`
- `createEventDispatcher` to callback props

This is the largest behavioral change — review the diff carefully and test.

## 3. Migrate SvelteKit 1 -> SvelteKit 2

```sh
npx sv migrate sveltekit-2
```

Updates imports, removed APIs, and config keys for the SvelteKit 2 breaking changes.

## 4. Migrate `$app/stores` -> `$app/state` (SvelteKit 2.12+)

```sh
npx sv migrate app-state
```

`$page` and `$navigating` from `$app/stores` become the reactive `$page` / `navigating` exports from `$app/state` — used with `$derived` in Svelte 5.

## 5. Self-closing tags in `.svelte` files

```sh
npx sv migrate self-closing-tags
```

Replaces self-closing non-void elements (e.g. `<div />`) with their explicit form (`<div></div>`). See sveltejs/kit#12128.

## 6. Migrate a library using `@sveltejs/package` v1 -> v2

```sh
npx sv migrate package
```

For library authors — updates the build configuration and output layout for `svelte-package` v2. See sveltejs/kit#8922.

## 7. Migrate pre-release SvelteKit -> filesystem routing

```sh
npx sv migrate routes
```

For projects on the SvelteKit pre-release `routes/` convention to move into `src/routes/`. See sveltejs/kit#5774.

## 8. Migrate Svelte 3 -> Svelte 4

```sh
npx sv migrate svelte-4
```

A much smaller migration than svelte-5. Updates deprecated APIs (slot props, `<svelte:fragment>` etc.).

---

## Common workflows

### Sequential migrations (Svelte 3 -> 5)

```sh
git commit -am "pre-migration"
npx sv migrate svelte-4
git commit -am "svelte 4"
npx sv migrate svelte-5
git commit -am "svelte 5"
```

### SvelteKit + Svelte combined upgrade

```sh
git commit -am "pre-upgrade"
npx sv migrate sveltekit-2
npx sv migrate svelte-5
npx sv migrate app-state
git diff
```

### Find manual follow-ups

```sh
rg "@migration" -t svelte -t ts -t js
```

Then search the docs referenced by each comment and complete the change.