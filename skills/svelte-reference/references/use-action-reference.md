# `use:` Action Reference

Complete reference for the `use:` directive and the `Action` / `ActionReturn` types from `svelte/action`.

> [!NOTE]
> In Svelte 5.29 and newer, consider using [attachments](@attach) instead, as they are more flexible and composable.

---

## Overview

```js
import type { Action, ActionReturn } from 'svelte/action';
```

Actions are functions called when an element is mounted. They typically use an `$effect` so they can reset state when the element unmounts.

```svelte
<script>
  /** @type {import('svelte/action').Action} */
  function myaction(node) {
    // setup runs on mount
    return {
      destroy() {
        // teardown runs on unmount
      }
    };
  }
</script>

<div use:myaction>...</div>
```

---

## `Action` Type

```ts
// @noErrors
Action<
//   1. node type (Element, HTMLElement, HTMLDivElement, ...)
//   2. parameter type (undefined if action takes no params)
//   3. custom event attributes (optional)
// > = (
//   node: Node,
//   parameter: Parameter
// ) => void | ActionReturn<Parameter, Events>;
```

### Type Parameters

| # | Name | Default | Description |
|---|------|---------|-------------|
| 1 | `Node` | `Element` | The element type. Use `Element` for "applies to anything". |
| 2 | `Parameter` | `undefined` | Type of the value passed via `use:action={value}`. |
| 3 | `Events` | `{}` | Map of `on*` event attributes the action exposes. |

---

## `ActionReturn` Type

The function may return either nothing, an `ActionReturn`, or — for legacy code — an object with `update`/`destroy`.

```ts
// @noErrors
ActionReturn<
  // 1. parameter type (for `update`)
  // 2. custom event attributes
> = {
  update?: (parameter: Parameter) => void;
  destroy?: () => void;
};
```

The `$effect` form is preferred — `update` is only needed for legacy migration.

---

## When Does Each Callback Run?

| Callback | Triggered when |
|----------|----------------|
| action body (mount code) | element is inserted into the DOM (client only — not SSR) |
| `update(newParam)` | (legacy) the parameter changes |
| `destroy()` | element is removed from the DOM |
| `$effect` setup | element is inserted |
| `$effect` teardown (returned cleanup) | element is removed |
| `$effect` re-run | any reactive dependency inside the effect changes (including the parameter, when read) |

The action body runs **once**. It does **not** re-run when the parameter changes.

---

## Re-running on Parameter Change

Two patterns:

### Pattern A — `$effect` reads the parameter

```svelte
<script>
  /** @type {import('svelte/action').Action<HTMLInputElement, number>} */
  function debounce(node, delay) {
    $effect(() => {
      // delay is read here — captured as a dependency
      const handler = () => setTimeout(() => node.dispatchEvent(new Event('done')), delay);
      node.addEventListener('input', handler);
      return () => {
        node.removeEventListener('input', handler);
      };
    });
  }
</script>

<input use:debounce={300} />
```

When `delay` changes, the previous effect teardown runs (removes old listener), then a new setup runs (adds new listener with new delay).

### Pattern B — Legacy `update` callback

```svelte
<script>
  /** @type {import('svelte/action').Action<HTMLDivElement, string>} */
  function colorize(node, color) {
    node.style.color = color;
    return {
      update(newColor) {
        node.style.color = newColor;
      },
      destroy() {
        node.style.color = '';
      }
    };
  }
</script>

<div use:colorize={'red'}>...</div>
```

> [!LEGACY]
> Prior to the `$effect` rune, actions could return an object with `update` and `destroy` methods. Using effects is preferred.

---

## Typing Custom Events

The third type parameter declares custom `on*` event attributes the action may dispatch:

```svelte
<script lang="ts">
  import type { Action } from 'svelte/action';

  type SwipeEvents = {
    onswipeleft: (e: CustomEvent<{ direction: 'left' }>) => void;
    onswiperight: (e: CustomEvent<{ direction: 'right' }>) => void;
  };

  const gestures: Action<HTMLDivElement, undefined, SwipeEvents> = (node) => {
    // dispatch new CustomEvent('swipeleft', { detail: { direction: 'left' } });
  };
</script>

<div use:gestures onswipeleft={next} onswiperight={prev}>
  swipe me
</div>
```

Events must be `CustomEvent<T>` — dispatch via `node.dispatchEvent(new CustomEvent('swipeleft', { detail }))`.

---

## Server-Side Rendering

Actions are **not** invoked during SSR. They only run on the client when the element is mounted.

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Reading `param` directly outside `$effect` | Wrap reads in `$effect` so changes trigger re-run |
| Returning `update` from an effect-style action | Don't — `$effect` already re-runs on dependency changes |
| Expecting `update` callback in modern code | Use `$effect` inside the action body instead |
| Forgetting `destroy` returns cleanup | Without it, listeners/observers leak across re-runs |

---

## Full Generic Example

```svelte
<script lang="ts" generics="E extends HTMLElement, T">
  import type { Action } from 'svelte/action';

  /** @type {Action<E, T>} */
  function identity(node, value) {
    // Read `value` inside an effect so re-runs happen on change
    $effect(() => {
      // ...use value...
    });
    return {
      destroy() {
        // ...
      }
    };
  }
</script>
```

---

## Related

- [examples/use-action.md](../examples/use-action.md) — 12 worked examples
- [svelte/action module docs](https://svelte.dev/docs/svelte/svelte-action)
- [attachments](@attach) — preferred in Svelte 5.29+