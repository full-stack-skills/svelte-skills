# Svelte 5 Best Practices Examples

Hands-on examples of recommended patterns from Svelte's official "Best practices" guide: `$state`, `$derived`, `$effect`, `$props`, `$inspect.trace`, events, snippets, each blocks, CSS variables, child component styling, context, async, and avoiding legacy features.

---

## 1. Use `$state` Only for Reactive Values

A variable doesn't need to be reactive unless it triggers an `$effect`, `$derived`, or template expression. Use plain `let` for non-reactive values to keep the code clean:

```svelte
<script>
  // Reactive — used in template
  let count = $state(0);

  // Not reactive — just a constant
  const TAX_RATE = 0.1;

  // Derived from state — re-evaluates when count changes
  const total = $derived(count * (1 + TAX_RATE));
</script>

<button onclick={() => count++}>{total.toFixed(2)}</button>
```

---

## 2. Use `$state.raw` for Reassigned-Only Large Objects

Proxying large objects for fine-grained reactivity is expensive. If you reassign rather than mutate (typical of API responses), use `$state.raw`:

```svelte
<script lang="ts">
  interface User { id: number; name: string; /* ...lots of fields */ }

  let user = $state.raw<User | null>(null);

  async function loadUser() {
    const res = await fetch('/api/me');
    user = await res.json(); // whole-object replacement, no deep proxy
  }
</script>

<button onclick={loadUser}>Load</button>
{#if user}<p>{user.name}</p>{/if}
```

---

## 3. Prefer `$derived` over `$effect` for Computation

The compiler can optimize `$derived` more than `$effect`. Use `$effect` only for side effects.

```svelte
<script>
  let num = $state(0);

  // DO
  let square = $derived(num * num);

  // DON'T
  // let square;
  // $effect(() => { square = num * num; });
</script>

<input type="number" bind:value={num} />
<p>{num}² = {square}</p>
```

When the expression is complex, use `$derived.by(() => ...)`:

```svelte
<script>
  let items = $state([1, 2, 3, 4, 5]);
  const stats = $derived.by(() => {
    const sum = items.reduce((a, b) => a + b, 0);
    return { sum, avg: sum / items.length, max: Math.max(...items) };
  });
</script>

<p>Sum: {stats.sum}, Avg: {stats.avg}, Max: {stats.max}</p>
```

> Deriveds are **writable** — assign to them, they re-evaluate when their expression changes. If the expression returns an object/array, it is returned as-is (not made reactive). Use `$state` inside `$derived.by` if you need reactivity in the result.

---

## 4. Treat `$effect` as an Escape Hatch — Avoid for State Updates

Effects are for syncing with the outside world, not for chaining state updates. Most of the time there's a better tool.

```svelte
<script>
  import { onMount } from 'svelte';
  let count = $state(0);

  // GOOD: log for debugging — use $inspect
  $inspect(count);

  // BAD: doing it in an effect
  // $effect(() => console.log(count));
</script>

<button onclick={() => count++}>{count}</button>
```

Prefer the targeted tool:

- **Sync to external library (D3, etc.)** → `{@attach ...}`
- **React to user input** → handler function or function binding
- **Log for debugging** → `$inspect` / `$inspect.trace`
- **Observe something outside Svelte** → `createSubscriber`

> Never wrap effect contents in `if (browser) {...}`. Effects don't run on the server at all.

---

## 5. Treat Props as Though They Will Change

Values that depend on props should be derived so they update if the prop changes:

```svelte
<script>
  let { type } = $props();

  // DO
  let color = $derived(type === 'danger' ? 'red' : 'green');

  // DON'T — `color` won't update if `type` changes
  // let color = type === 'danger' ? 'red' : 'green';
</script>

<p style:color>{type}</p>
```

---

## 6. Use `$inspect.trace` to Debug Reactivity

When a value isn't updating (or is updating too often), add `$inspect.trace` as the first line to log the dependency tree:

```svelte
<script>
  let { userId } = $props();
  let user = $state(null);

  $effect(() => {
    $inspect.trace('user-load');
    fetch(`/api/users/${userId}`).then(r => r.json()).then(u => user = u);
  });
</script>
```

Open the console and click into the logged tree to see which dependency triggered which update.

---

## 7. Events — Use Attribute Syntax, Not `on:`

Any attribute starting with `on` is an event listener. Shorthand and spread both work:

```svelte
<script>
  function handleClick() { /* ... */ }
  let { ...rest } = $props();
</script>

<button onclick={handleClick}>click me</button>

<!-- shorthand -->
<button {onclick}>...</button>

<!-- spread -->
<button {...rest}>...</button>
```

For `window` and `document` listeners, prefer `<svelte:window>` / `<svelte:document>`:

```svelte
<svelte:window onkeydown={handleKey} />
<svelte:document onvisibilitychange={handleVisibility} />
```

> Don't use `onMount` or `$effect` to attach to `window`/`document` — the built-in elements handle teardown automatically.

---

## 8. Snippets for Reusable Markup

Snippets live in the template and can be rendered with `{@render}` or passed as props:

```svelte
{#snippet greeting(name)}
  <p>hello {name}!</p>
{/snippet}

{@render greeting('world')}
```

A snippet that doesn't reference component state can be exported from a `<script module>`:

```svelte
<script module>
  export const formatDate = (d) => d.toLocaleDateString();
</script>
```

> Top-level snippets (outside elements/blocks) can be referenced from `<script>`.

---

## 9. Keyed Each Blocks for Performance

Always key by a stable, unique property — never the index:

```svelte
<script>
  let { items } = $props();
</script>

<ul>
  {#each items as item (item.id)}
    <li>{item.name}</li>
  {/each}
</ul>
```

Avoid destructuring inside the each when you need to mutate the item (e.g. with `bind:value={item.count}`):

```svelte
<!-- GOOD: item is stable across renders -->
{#each items as item (item.id)}
  <input bind:value={item.count} />
{/each}
```

---

## 10. JS Variables in CSS via `style:--`

Use the `style:` directive to set a CSS custom property; then reference `var(--name)` inside `<style>`:

```svelte
<script>
  let { columns = 3 } = $props();
</script>

<div style:--columns={columns} class="grid">
  {#each Array(9) as _, i}
    <div>{i + 1}</div>
  {/each}
</div>

<style>
  .grid {
    display: grid;
    grid-template-columns: repeat(var(--columns), 1fr);
    gap: 1rem;
  }
</style>
```

---

## 11. Styling Child Components with CSS Variables

A child's CSS is scoped to itself. The clean way for the parent to influence the child is via CSS custom properties:

```svelte
<!-- Parent.svelte -->
<Child --color="red" --size="2rem" />
```

```svelte
<!-- Child.svelte -->
<h1>Hello</h1>

<style>
  h1 {
    color: var(--color, inherit);
    font-size: var(--size, 1rem);
  }
</style>
```

When the child is from a library and you can't use variables, fall back to `:global`:

```svelte
<style>
  div :global {
    h1 {
      color: red;
    }
  }
</style>
```

---

## 12. Prefer `createContext` over Module-Level Shared State

Context scopes state to the part of the app that needs it and doesn't leak between users during SSR:

```ts
// theme.ts
import { createContext } from 'svelte';

export const [getTheme, setTheme] = createContext<{
  mode: 'light' | 'dark';
  toggle: () => void;
}>();
```

```svelte
<!-- Root.svelte -->
<script>
  import { setTheme } from './theme';
  let mode = $state<'light' | 'dark'>('light');
  setTheme({
    mode,
    toggle: () => { mode = mode === 'light' ? 'dark' : 'light'; }
  });
</script>
```

```svelte
<!-- DeepChild.svelte -->
<script>
  import { getTheme } from './theme';
  const theme = getTheme();
</script>

<button onclick={theme.toggle}>
  Current mode: {theme.mode}
</button>
```

`createContext` is type-safe — the value passed to `setTheme` must match the type, and `getTheme` returns that type.

---

## 13. Async Svelte (await + hydratable, 5.36+)

Svelte 5.36+ supports awaiting promises directly in templates, plus `hydratable` for SSR data reuse. Enable the `experimental.async` option in `svelte.config.js`:

```js
// svelte.config.js
export default {
  compilerOptions: {
    experimental: { async: true }
  }
};
```

```svelte
<script>
  async function getPosts() {
    const res = await fetch('/api/posts');
    return res.json();
  }
  const postsPromise = getPosts();
</script>

<h1>Posts</h1>
{#await postsPromise}
  <p>Loading…</p>
{:then posts}
  <ul>{#each posts as p (p.id)}<li>{p.title}</li>{/each}</ul>
{:catch err}
  <p>Error: {err.message}</p>
{/await}
```

---

## 14. Avoid Legacy Features

Always use the modern replacements in new code:

| Legacy | Replacement |
|--------|-------------|
| `let count = 0; count++` (implicit reactivity) | `$state(0)` |
| `$:` assignments and statements | `$derived` / `$effect` |
| `export let prop` / `$$props` / `$$restProps` | `$props()` |
| `on:click={fn}` | `onclick={fn}` |
| `<slot>` / `$$slots` / `<svelte:fragment>` | `{#snippet}` + `{@render}` |
| `<svelte:component this={C}>` | `<C>` (or `<svelte:element this={C}>`) |
| `<svelte:self>` | `import Self from './Self.svelte'; <Self />` |
| Stores | Classes with `$state` fields |
| `use:action` | `{@attach ...}` |
| `class:` directive | clsx-style arrays/objects in `class` |

Example: from `class:` to clsx-style:

```svelte
<!-- LEGACY -->
<button class:primary class:large class:disabled={isDisabled}>click</button>

<!-- MODERN -->
<script>
  import { clsx } from 'clsx'; // or hand-rolled
  let { variant, size, isDisabled } = $props();
</script>
<button class={clsx(variant, size, { disabled: isDisabled })}>click</button>
```
