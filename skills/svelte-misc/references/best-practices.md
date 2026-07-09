# Svelte 5 Best Practices Reference

A comprehensive reference to the recommended patterns from Svelte's official "Best practices" guide. Use this as a checklist when designing new code.

---

## `$state`

Use `$state` only when reactivity is needed (a value that triggers an `$effect`, `$derived`, or template expression). Everything else can be a plain `let`.

### Deep vs Raw

- `$state(obj)` and `$state(array)` proxy the value for fine-grained reactivity — mutation triggers updates.
- `$state.raw` does not proxy. Use it when you **reassign** rather than **mutate** (typical for API responses, large static records).

```js
let largeResponse = $state.raw(null);   // no proxy overhead
largeResponse = await fetch(...).then(r => r.json());
```

### Trade-offs

- Proxy overhead: large objects initialized once and reassigned should be `$state.raw`.
- Mutation APIs: objects you mutate (forms, drag-and-drop models) should remain proxied.
- `$state` inside `$derived.by` is fine — `$derived`'s expression itself returns the raw value.

---

## `$derived`

Prefer `$derived` over `$effect` for pure computation. `$derived` returns the latest value eagerly; effects are scheduled.

### Expression vs `by`

```js
// expression — use when it fits on one line
let fullName = $derived(`${first} ${last}`);

// .by — use when logic is multi-step
let stats = $derived.by(() => {
  const sum = items.reduce((a, b) => a + b, 0);
  return { sum, avg: sum / items.length };
});
```

### Writable

Derived values are writable. Assignment forces a re-evaluation based on the new expression value, NOT a recompute via the function:

```js
let total = $derived(count * price);
total = 0; // valid assignment, not derived behavior
```

### Object/Array Results

If the expression returns an object or array, the result is **not** made deeply reactive. Use `$state` inside `$derived.by` if you need reactivity in the result.

---

## `$effect`

Effects are an escape hatch for syncing with the **outside world**. Treat them as a last resort.

### Better Alternatives

| Need | Better tool |
|------|-------------|
| Sync to external library (D3, charting) | `{@attach ...}` |
| Respond to user input | inline event handler / function binding |
| Log for debugging | `$inspect` / `$inspect.trace` |
| Observe something external | `createSubscriber` |
| Compute from other state | `$derived` |

### Server-Side Behaviour

Effects do not run on the server. Don't wrap effect contents in `if (browser) {...}` — they never run on the server in the first place.

### Reading vs Writing

Avoid updating state from inside an effect; prefer to compute with `$derived`. Effects writing state cause update loops that are hard to reason about.

---

## `$props`

Treat props as though they will change. Anything dependent on a prop should be `$derived`:

```js
let { type } = $props();
let color = $derived(type === 'danger' ? 'red' : 'green'); // updates if `type` changes
```

Plain reads (`let color = type === 'danger' ? 'red' : 'green'`) freeze on first render and won't react to prop updates.

---

## `$inspect.trace`

When a value isn't updating correctly (or updates too often), add `$inspect.trace(label)` as the first line of the relevant `$effect` or `$derived.by`:

```js
$effect(() => {
  $inspect.trace('load-user');
  loadUser(userId);
});
```

It traces dependencies and shows which dependency triggered which update in the console.

---

## Events

Any attribute starting with `on` is an event listener. The shorthand and the spread attribute both work:

```svelte
<button onclick={handle}>click</button>
<button {onclick}>...</button>
<button {...props}>...</button>
```

For `window` and `document` listeners, use `<svelte:window>` and `<svelte:document>`:

```svelte
<svelte:window onkeydown={handle} />
<svelte:document onvisibilitychange={vis} />
```

Avoid `onMount` / `$effect` for window/document — Svelte handles teardown automatically with the dedicated elements.

---

## Snippets

Snippets are reusable markup pieces declared in the template:

```svelte
{#snippet greeting(name)}
  <p>hello {name}!</p>
{/snippet}

{@render greeting('world')}
```

A snippet that doesn't reference component state can be exported from a `<script module>` and reused outside.

---

## Each Blocks

Prefer **keyed** each blocks. The key must be a unique, stable identifier of the item — never the index:

```svelte
{#each items as item (item.id)}
  <Item {item} />
{/each}
```

Avoid destructuring the each item if you need to mutate it (e.g. `bind:value={item.count}`):

```svelte
<!-- BAD: destructure severs the binding back to the array -->
{#each items as { id, ...rest } (id)} <Item {...rest} /> {/each}

<!-- GOOD: keep the item reference -->
{#each items as item (item.id)} <Item {item} /> {/each}
```

---

## CSS Variables from JS

Set a CSS custom property with `style:--`:

```svelte
<div style:--columns={columns}>...</div>

<style>
  .grid {
    grid-template-columns: repeat(var(--columns), 1fr);
  }
</style>
```

---

## Styling Child Components

CSS in a component's `<style>` is scoped to that component. To let a parent influence child styles:

1. **Best** — pass CSS custom properties:

```svelte
<Child --color="red" --size="2rem" />
```

```svelte
<!-- Child.svelte -->
<style>
  h1 { color: var(--color); font-size: var(--size); }
</style>
```

2. **Fallback** — use `:global` to pierce scoping (only when CSS variables aren't possible, e.g. child is from a library):

```svelte
<style>
  div :global {
    h1 { color: red; }
  }
</style>
```

---

## Context

Prefer `createContext` over a shared module with state. Context scope prevents leak between users during SSR.

`createContext` provides type safety — both `setContext` and `getContext` are typed.

```ts
import { createContext } from 'svelte';
export const [getTheme, setTheme] = createContext<Theme>();
```

---

## Async Svelte (Experimental)

Svelte 5.36+ supports awaiting promises inside templates plus `hydratable` for SSR data reuse. Enable the `experimental.async` compilerOption:

```js
// svelte.config.js
export default {
  compilerOptions: { experimental: { async: true } }
};
```

These APIs are not yet fully stable; check the changelog before relying on them in production.

---

## Replace Legacy Features in New Code

| Legacy | Replacement |
|--------|-------------|
| `let count = 0; count++` (top-level implicit reactivity) | `$state(0)` |
| `$:` statements | `$derived` / `$effect` |
| `export let prop` / `$$props` / `$$restProps` | `$props()` |
| `on:click={fn}` | `onclick={fn}` |
| `<slot>` / `$$slots` / `<svelte:fragment>` | `{#snippet}` + `{@render}` |
| `<svelte:component this={C}>` | `<C>` (or `<svelte:element this={C}>`) |
| `<svelte:self>` | `import Self from './Self.svelte'; <Self />` |
| Stores | Classes with `$state` fields |
| `use:action` | `{@attach ...}` |
| `class:foo` directive | clsx-style array/object in `class` |

---

## Testing Notes

Svelte is unopinionated about test frameworks. Recommended:

- **Unit / Integration** — Vitest (works out of the box with Vite, supports runes in `.svelte.ts`)
- **Component** — `@testing-library/svelte` (or raw `mount`/`unmount` from `svelte`)
- **Storybook** — for story-based component testing
- **E2E** — Playwright / Cypress / Nightwatch

For runes in tests: filename must include `.svelte` (e.g. `multiplier.svelte.test.js`). For tests involving `$effect`, wrap them in `$effect.root(() => { ... })` and call the returned cleanup function.

---

## Quick Checklist

- [ ] Used `$state` only where reactivity is needed
- [ ] Used `$state.raw` for large, reassigned-only objects
- [ ] Used `$derived` (or `$derived.by`) instead of `$effect` for computation
- [ ] Used `$effect` only as a last resort and avoided state updates inside it
- [ ] Derived values that depend on props
- [ ] Used `$inspect.trace` when reactivity was unclear
- [ ] Event handlers use attribute syntax (`onclick`, not `on:click`)
- [ ] `<svelte:window>` / `<svelte:document>` for global listeners
- [ ] Keyed each blocks with stable identifiers (never index)
- [ ] CSS variables for cross-component styling instead of `:global` when possible
- [ ] `createContext` for shared state instead of module-level singletons
- [ ] No legacy syntax (`on:`, `export let`, `$:`, `<slot>`) in new code
