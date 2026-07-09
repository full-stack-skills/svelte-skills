# `animate:` Directive Reference

Complete reference for the `animate:` directive and `svelte/animate`.

---

## What `animate:` Does

`animate:` runs when the **index of an existing item changes** inside a [keyed each block](https://svelte.dev/docs/svelte/each#Keyed-each-blocks). It plays a one-shot animation that moves the element from its old position to its new one.

> **Important:** animations do **not** run when an element is added or removed — only when an existing item is **reordered**. Use `in:` / `out:` for add/remove.

```svelte
<script>
  import { flip } from 'svelte/animate';
  let list = $state([1, 2, 3, 4, 5]);
</script>

<!-- When `list` is reordered the animation will run -->
{#each list as item (item)}
  <li animate:flip>{item}</li>
{/each}
```

---

## Rules

1. **`animate:` must be on an immediate child of a keyed `{#each ... (key)}` block.**
   The `key` is required — without it, Svelte can't know which element moved.

   ```svelte
   <!-- RIGHT -->
   {#each items as item (item.id)}
     <div animate:flip>{item.text}</div>
   {/each}

   <!-- WRONG: animate: is not on the direct child -->
   {#each items as item (item.id)}
     <div>
       <span animate:flip>{item.text}</span>  <!-- does NOT animate -->
     </div>
   {/each}
   ```

2. **`animate:` only runs on reorder** — not on add/remove. Compose with `in:` / `out:` for the latter.

3. **Key must be stable** — use a unique ID, never the array index.

---

## Module Import

```js
import { flip } from 'svelte/animate';
```

Only `flip` is built-in. For anything else, write a custom animation function.

---

## `flip`

The FLIP technique (First, Last, Invert, Play):

1. **F**irst — record starting position (`from: DOMRect`)
2. **L**ast — record ending position (`to: DOMRect`)
3. **I**nvert — apply a `transform` that makes the element *look like* it's still at `from`
4. **P**lay — animate the `transform` from inverse back to identity

```ts
flip(node: HTMLElement, params?: {
  delay?: number;
  duration?: number | ((node: HTMLElement, { from, to }) => number);
  easing?: (t: number) => number;
}): TransitionConfig;
```

```svelte
{#each items as item (item)}
  <li animate:flip={{ duration: 400, easing: quintOut }}>{item}</li>
{/each}
```

### Dynamic `duration`

Pass a function — useful for distance-aware timing:

```js
flip(node, { from, to }, params) {
  return {
    duration: Math.sqrt(
      (from.left - to.left) ** 2 + (from.top - to.top) ** 2
    ) * 120
  };
}
```

---

## Custom Animation Functions

```ts
// @noErrors
animation = (
  node: HTMLElement,
  { from: DOMRect, to: DOMRect },
  params: any
) => {
  delay?: number;
  duration?: number;
  easing?: (t: number) => number;
  css?: (t: number, u: number) => string;   // preferred — runs off main thread
  tick?: (t: number, u: number) => void;    // imperative fallback
};
```

- `from` — bounding rect **before** the reorder
- `to` — bounding rect **after** the reorder
- `t` — eased progress, `0 → 1`
- `u` — `1 - t`

### CSS-based (preferred)

```svelte
<script>
  import { cubicOut } from 'svelte/easing';

  /**
   * @param {HTMLElement} node
   * @param {{ from: DOMRect; to: DOMRect }} states
   * @param {any} _params
   */
  function whizz(node, { from, to }, _params) {
    const dx = from.left - to.left;
    const dy = from.top - to.top;
    const distance = Math.sqrt(dx * dx + dy * dy);

    return {
      delay: 0,
      duration: Math.sqrt(distance) * 120,
      easing: cubicOut,
      css: (t, u) =>
        `transform: translate(${u * dx}px, ${u * dy}px) rotate(${t * 360}deg);`
    };
  }
</script>

{#each list as item (item)}
  <div animate:whizz>{item}</div>
{/each}
```

### Tick-based (imperative)

Use only when CSS can't express it (e.g. color + transform simultaneously, or text mutation):

```svelte
<script>
  import { cubicOut } from 'svelte/easing';

  function colorFlip(node, _states) {
    return {
      duration: 600,
      easing: cubicOut,
      tick: (t) => {
        node.style.color = t > 0.5 ? 'Pink' : 'Cyan';
        node.style.transform = `scale(${0.8 + 0.2 * t})`;
      }
    };
  }
</script>

{#each items as item (item)}
  <li animate:colorFlip>{item}</li>
{/each}
```

> [!NOTE] If it's possible to use `css` instead of `tick`, do so — web animations can run off the main thread, preventing jank on slower devices.

---

## Parameters

```ts
{
  delay?: number;                                 // ms before starting
  duration?: number | (node, { from, to }) => number;  // total time
  easing?: (t: number) => number;                 // easing function
  css?: (t: number, u: number) => string;         // CSS-based
  tick?: (t: number, u: number) => void;          // imperative
}
```

`css` and `tick` are mutually exclusive.

---

## Animation vs Transition

| | `animate:` | `transition:` / `in:` / `out:` |
|---|---|---|
| Trigger | keyed each block **reorder** | element **enter** or **leave** DOM |
| Position info | `from` / `to` `DOMRect` | none |
| Reversible | no | `transition:` yes; `in:` / `out:` no |
| Keyed each? | required | not required |
| Where on tree | immediate child of `{#each ... (key)}` | any element |
| Typical use | list shuffling, sorting | modals, toasts, expanding panels |

---

## Common Patterns

### Reorderable list (sortable / drag-to-reorder)

```svelte
{#each items as item (item.id)}
  <li animate:flip={{ duration: 300 }}>{item.text}</li>
{/each}
```

### Add/remove + reorder composed

```svelte
{#each items as item (item.id)}
  <li
    in:fade
    out:slide
    animate:flip={{ duration: 300 }}
  >{item.text}</li>
{/each}
```

### Distance-aware rotation

```svelte
<script>
  import { cubicOut } from 'svelte/easing';

  function whizz(node, { from, to }) {
    const dx = from.left - to.left;
    const dy = from.top - to.top;
    return {
      duration: 600,
      easing: cubicOut,
      css: (t, u) => `transform: translate(${u * dx}px, ${u * dy}px) rotate(${t * 360}deg);`
    };
  }
</script>

{#each items as item (item)}
  <div animate:whizz>{item}</div>
{/each}
```

---

## Gotchas

| Issue | Cause / Fix |
|-------|-------------|
| `animate:` does nothing | Missing `(key)` on `{#each}` — the each block must be **keyed** |
| `animate:` does nothing on add/remove | That's the rule — use `in:` / `out:` instead |
| Animation only plays on direct child | `animate:` must be on the immediate child of the keyed each block; move the directive up the tree if you want a larger container to slide |
| Jumpy animation | Check that the key is unique and stable (not the array index) |
| Type errors on custom animation | Annotate the function: `/** @type {(node: HTMLElement, states: { from: DOMRect; to: DOMRect }, params: any) => import('svelte/animate').AnimationConfig} */` |

---

## Related

- [examples/in-out-animate.md](../examples/in-out-animate.md) — 12 worked examples
- [examples/transition.md](../examples/transition.md) — basic transitions
- [references/transitions-complete.md](./transitions-complete.md) — full built-in transition API
- [references/in-out-reference.md](./in-out-reference.md) — `in:` / `out:` separate transitions
