# `in:` and `out:` Reference

Complete reference for the `in:` and `out:` directives — separate, unidirectional transitions.

---

## Why `in:` / `out:` instead of `transition:`?

`transition:` is **bidirectional**: if you toggle a block off mid-intro, Svelte *reverses* the in-progress animation. `in:` and `out:` are **unidirectional**:

| Directive | When it runs | Reversible on interrupt? |
|-----------|--------------|--------------------------|
| `transition:` | entering or leaving | **yes** — reverses mid-flight |
| `in:` | entering only | **no** — if interrupted, the in-transition is **abandoned** and the out-transition restarts from `t=0` |
| `out:` | leaving only | **no** — plays forward to completion |

Use `in:` / `out:` when enter and exit should be visually different animations, or when the exit should *not* reverse the entry.

---

## Directive Syntax

```svelte
<!-- enter only -->
<div in:fade>...</div>

<!-- leave only -->
<div out:fade>...</div>

<!-- with parameters -->
<div in:fly={{ y: 200, duration: 300 }} out:fade={{ duration: 200 }}>
  ...
</div>

<!-- both as separate directives on the same element -->
<div in:fly={{ y: 50 }} out:slide={{ duration: 250 }}>
  ...
</div>

<!-- combine with |global -->
<div in:fade|global out:fly|global>...</div>
```

The double `{{curlies}}` are not special syntax — they are an object literal inside an expression tag.

---

## Semantics

```svelte
<script>
  import { fade, fly } from 'svelte/transition';
  let visible = $state(false);
</script>

<label>
  <input type="checkbox" bind:checked={visible} />
  visible
</label>

{#if visible}
  <div in:fly={{ y: 200 }} out:fade>flies in, fades out</div>
{/if}
```

- On intro: `fly` plays from `0 → 1` over 200px.
- On outro: `fade` plays from `1 → 0`.
- If the block is toggled off *while* `fly` is still playing, the in-flight `fly` is cancelled and `fade` starts from `t=0`.

> Key difference from `transition:fly`: with `in:fly` + `out:fade`, the **out-transition does not reverse the in-transition** — they're independent.

---

## `options` Argument

A transition function receives a third argument with `direction`:

```ts
transition = (node, params, options: { direction: 'in' | 'out' | 'both' }) => {...}
```

`direction` lets the same function be used for both `in:` and `out:` (or as a `transition:`):

```js
/// @noErrors
function fade(node, { duration = 400 }, { direction }) {
  // direction is 'in' or 'out' depending on the directive
  return {
    duration,
    css: (t) => `opacity: ${direction === 'in' ? t : 1 - t}`
  };
}
```

---

## Events on `in:` / `out:` Elements

The standard transition events (`introstart`, `introend`, `outrostart`, `outroend`) fire as usual — see [references/transitions-complete.md](./transitions-complete.md#transition-events).

---

## When to Choose `in:` / `out:` Over `transition:`

| Use case | Choose |
|----------|--------|
| Same animation for enter and exit (reversible) | `transition:` |
| Enter and exit are visually different | `in:` + `out:` |
| Out should NOT be the reverse of in | `in:` + `out:` |
| You want one-sided (e.g. fade in only, no exit animation) | `in:` (or `out:`) alone |
| Exit should not be a smooth reverse — abrupt | `out:` with short/no `duration` |

---

## Combining `in:` / `out:` With `animate:`

In a keyed each block, `in:` and `out:` play for added/removed items, and `animate:flip` plays for reorders. They compose cleanly:

```svelte
{#each items as item (item.id)}
  <li in:fade out:slide animate:flip={{ duration: 300 }}>
    {item.text}
  </li>
{/each}
```

See [references/animations-reference.md](./animations-reference.md) for `animate:`.

---

## Common Patterns

### Asymmetric dismiss (fly in, fade out)

```svelte
<div in:fly={{ y: 50, duration: 300 }} out:fade={{ duration: 200 }}>
  toast
</div>
```

### Staggered (same transition, out delayed)

```svelte
<div in:fade={{ delay: 0 }} out:fade={{ delay: 200, duration: 800 }}>
  polite dismissal
</div>
```

### Snappy in, slow out

```svelte
<div in:scale={{ start: 0.9, duration: 150 }} out:fade={{ duration: 600 }}>
  ...
</div>
```

### No exit (just appear)

```svelte
<div in:fly={{ y: 20, duration: 250 }}>
  no exit animation — disappears instantly when block is removed
</div>
```

---

## Built-in `in:` / `out:` Usage

All built-in transitions from `svelte/transition` (`fade`, `fly`, `slide`, `scale`, `blur`, `draw`, `crossfade`) work with `in:` and `out:`:

```svelte
<script>
  import { fade, fly, scale, slide } from 'svelte/transition';
</script>

{#if x}
  <div in:scale out:slide>scale in, slide out</div>
  <div in:fly={{ y: 30 }} out:fade={{ duration: 500 }}>fly in, slow fade out</div>
{/if}
```

For full per-transition parameter tables, see [references/transitions-complete.md](./transitions-complete.md).

---

## Related

- [examples/in-out-animate.md](../examples/in-out-animate.md) — 12 worked examples
- [references/transitions-complete.md](./transitions-complete.md) — full built-in transition API
- [references/animations-reference.md](./animations-reference.md) — `animate:` directive
