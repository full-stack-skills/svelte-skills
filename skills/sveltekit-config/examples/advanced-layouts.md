# Advanced layouts

The default layout chain mirrors the folder chain. Use these patterns to reshape it.

## 1. Layout groups `(group)`

Parentheses don't appear in the URL — use to share a layout across routes.

```tree
src/routes/
├ (app)/
│  ├ dashboard/+page.svelte
│  ├ item/[id]/+page.svelte
│  └ +layout.svelte        # shell with sidebar
├ (marketing)/
│  ├ about/+page.svelte
│  ├ testimonials/+page.svelte
│  └ +layout.svelte        # shell with hero banner
└ +layout.svelte           # root: <html> boilerplate only
```

`/dashboard` → inherits root + `(app)`.
`/about` → inherits root + `(marketing)`.

## 2. Layout groups with shared root layout

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  let { children } = $props();
</script>
<html>
  <body>{@render children()}</body>
</html>
```

```svelte
<!-- src/routes/(app)/+layout.svelte -->
<script>
  import Sidebar from '$lib/Sidebar.svelte';
  let { children } = $props();
</script>
<Sidebar />
<main>{@render children()}</main>
```

## 3. Layout group with its own `+page`

You can put `+page.svelte` inside a group:

```tree
src/routes/
├ (app)/+page.svelte       # /  (uses (app) layout)
├ (app)/dashboard/+page.svelte
└ (app)/+layout.svelte
```

## 4. Breaking out of layouts — `+page@`

```tree
src/routes/
├ (app)/
│  ├ item/[id]/
│  │  ├ embed/+page.svelte   # inherits (app)/item/[id] chain
│  │  └ +layout.svelte
│  ├ +layout.svelte
├ +layout.svelte
```

For `embed` to skip the (app) layout (use only root + item layouts), rename:

```tree
src/routes/(app)/item/[id]/embed/+page@(app).svelte
```

Options:

| Filename | Inherits from |
|----------|---------------|
| `+page@[id].svelte` | `(app)/item/[id]/+layout.svelte` |
| `+page@item.svelte` | `(app)/item/+layout.svelte` |
| `+page@(app).svelte` | `(app)/+layout.svelte` |
| `+page@.svelte` | `src/routes/+layout.svelte` (root) |

## 5. Breaking out — `+layout@`

Layouts can also break out:

```tree
src/routes/
├ (app)/
│  ├ item/+layout@.svelte   # rewinds to root, skips (app)
│  ├ item/+page.svelte
│  └ +layout.svelte         # (app) layout, NOT inherited by item/
└ +layout.svelte
```

## 6. Outlier routes — don't use a group

```tree
src/routes/
├ (app)/...           # most routes
└ admin/+page.svelte  # exception: no (app) layout
```

`/admin` inherits only the root layout.

## 7. `+error.svelte` placement

```tree
src/routes/
├ +error.svelte        # catches errors from anywhere
└ (app)/
   └ +error.svelte     # catches errors specific to (app) routes
```

The closest `+error.svelte` wins.

## 8. Composition as alternative

When the group nesting gets ugly, just compose:

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

Use composition when layout grouping feels overengineered.

## 9. Conditional layout based on data

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import { page } from '$app/state';
  import AppShell from '$lib/AppShell.svelte';
  import MarketingShell from '$lib/MarketingShell.svelte';
  let { children } = $props();
</script>

{#if page.url.pathname.startsWith('/dashboard')}
  <AppShell>{@render children()}</AppShell>
{:else}
  <MarketingShell>{@render children()}</MarketingShell>
{/if}
```

Simpler than nested groups for small apps.

## 10. Per-page layout reset for full-screen pages

```tree
src/routes/
├ +layout.svelte            # standard app shell
├ +page.svelte
└ settings/
   ├ +layout.svelte         # standard sub-layout
   ├ +page.svelte
   └ fullscreen/+page@.svelte  # only root layout — no settings chrome
```

## See also

- [Layouts reference](../references/layouts-advanced-reference.md)
