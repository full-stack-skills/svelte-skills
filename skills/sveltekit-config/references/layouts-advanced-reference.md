# Advanced layouts reference

Reshape the default folder-mirrors-URL layout hierarchy using groups and `@` reset syntax.

## Default behavior

The layout chain mirrors the folder chain:

```tree
src/routes/
├ +layout.svelte              # root
├ +page.svelte                # uses root
├ admin/
│  ├ +layout.svelte           # root → admin
│  └ +page.svelte             # uses root + admin
└ admin/users/
   └ +page.svelte             # uses root + admin
```

## Route groups — `(group)`

Parentheses around a folder name keep it in the layout hierarchy but remove it from the URL.

```tree
src/routes/
├ (marketing)/
│  ├ +layout.svelte           # marketing shell
│  ├ +page.svelte             # /
│  ├ about/+page.svelte       # /about
│  └ pricing/+page.svelte     # /pricing
├ (app)/
│  ├ +layout.svelte           # app shell
│  ├ dashboard/+page.svelte   # /dashboard
│  └ settings/+page.svelte    # /settings
└ +layout.svelte              # root: html shell
```

`/` inherits `root + (marketing)`.
`/dashboard` inherits `root + (app)`.

### When to use groups

- Sharing a layout (sidebar, theme) across unrelated URL segments
- Excluding routes from a layout (put them outside the group)
- Multiple `+page.svelte` files at the same URL — impossible; groups are how you choose

### `+page` inside a group

A group can contain `+page.svelte`:

```tree
src/routes/(marketing)/+page.svelte   # /
```

## Breaking out — `+page@something.svelte`

Append `@<segment>` to reset the layout chain to a specific ancestor.

```tree
src/routes/
├ (app)/
│  ├ item/[id]/
│  │  ├ embed/+page.svelte   # default chain: root → (app) → item → [id]
│  │  └ +layout.svelte
│  ├ item/+layout.svelte
│  └ +layout.svelte
└ +layout.svelte
```

Rename `embed/+page.svelte` to one of:

| Filename | Inherits |
|----------|----------|
| `+page@[id].svelte` | `(app)/item/[id]/+layout.svelte` (chain breaks, only this layout) |
| `+page@item.svelte` | `(app)/item/+layout.svelte` (skip [id]) |
| `+page@(app).svelte` | `(app)/+layout.svelte` (skip item, [id]) |
| `+page@.svelte` | `+layout.svelte` (root only — skip everything) |

## Breaking out — `+layout@.svelte`

Layouts can also break out:

```tree
src/routes/(app)/
├ item/+layout@.svelte    # rewinds to root for everything below
├ item/[id]/+page.svelte  # uses root + item only
└ +layout.svelte
```

This is equivalent to moving `item/` out of `(app)`.

## Outlier pattern

Put most routes inside a group, exceptions outside:

```tree
src/routes/
├ (app)/dashboard/+page.svelte
├ (app)/+layout.svelte
└ admin/+page.svelte   # does NOT inherit (app)
```

`/admin` uses root only.

## Error layout boundaries

`+error.svelte` files catch errors from their segment and below. The closest `+error.svelte` wins.

```tree
src/routes/
├ +error.svelte              # catches all
└ (app)/
   ├ +error.svelte           # catches errors in (app) only
   └ +layout.svelte
```

Errors in `/dashboard` → `(app)/+error.svelte`.
Errors in `/about` → root `+error.svelte`.

Special case: if the error occurs in the **root** `+layout.js` / `+layout.server.js`, SvelteKit falls back to `src/error.html` (or a default error page) instead of rendering the root `+error.svelte`, since the root layout would contain it.

## Composition as alternative

Sometimes groups feel heavy. Compose instead:

```svelte
<!-- src/routes/nested/route/+layout@.svelte -->
<script>
  import ReusableLayout from '$lib/ReusableLayout.svelte';
  let { data, children } = $props();
</script>

<ReusableLayout {data}>
  {@render children()}
</ReusableLayout>
```

```js
// src/routes/nested/route/+layout.js
import { reusableLoad } from '$lib/reusable-load-function';
export const load = (event) => reusableLoad(event);
```

Use when:
- The grouping would be very deep
- You're building a single outlier
- You want a flexible, reusable layout component

## When NOT to use groups

- Simple conditional UI based on URL — use `{#if}` in the root layout
- One-off styling — use a CSS class on `<body>`
- Sharing between most routes — put the shared UI in the root layout

## Layout data flow

Layouts receive `data` from their own `+layout.js`/`+layout.server.js` and from descendant pages:

```svelte
<!-- +layout.svelte -->
<script>
  let { data, children } = $props();
  // data comes from this layout's load + all child pages' load
</script>
```

`+layout.server.js` and `+layout.js` can both exist; universal runs first, then server merges.

## See also

- [examples/advanced-layouts.md](../examples/advanced-layouts.md)
