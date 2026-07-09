# Transitions Complete Reference

Complete reference for `svelte/transition` and `transition:` directives.

---

## Module Import

```js
import {
  fade,
  blur,
  fly,
  slide,
  scale,
  draw,
  crossfade
} from 'svelte/transition';
```

---

## Directive Forms

```svelte
<!-- bidirectional (reverses on interrupt) -->
<div transition:fade>...</div>

<!-- enter-only -->
<div in:fly={{ y: 50 }}>...</div>

<!-- leave-only -->
<div out:fade>...</div>

<!-- global (fires when ancestor blocks change too) -->
<div transition:fade|global>...</div>

<!-- with parameters (object literal in expression tag) -->
<div transition:fade={{ duration: 300, delay: 100 }}>...</div>
```

The double `{{curlies}}` are not special syntax — they are an object literal inside an expression tag.

---

## Common Transition Parameters

Every built-in (and custom) transition accepts this base object:

```ts
{
  delay?: number;                                  // ms before starting (default 0)
  duration?: number;                                // ms duration (default 400)
  easing?: (t: number) => number;                   // easing function (default linear)
  css?: (t: number, u: number) => string;           // web animation (preferred)
  tick?: (t: number, u: number) => void;            // imperative callback
}
```

`t` runs from `0` to `1` on enter, `1` to `0` on leave. `u = 1 - t`. `t` already has easing applied by the time it reaches `css` / `tick`. `1` is the element's natural state.

**`css` and `tick` are mutually exclusive** — only one per transition.

---

## Built-in Transitions

### `fade`

Animate opacity `0 → 1` (intro) / `1 → 0` (outro).

```ts
fade(node: Element, params?: {
  delay?: number;
  duration?: number;
  easing?: (t: number) => number;
}): TransitionConfig;
```

```svelte
<div transition:fade={{ duration: 300 }}>fades in</div>
```

---

### `blur`

Animate `filter: blur(Npx)` plus opacity.

```ts
blur(node: Element, params?: {
  delay?: number;
  duration?: number;
  easing?: (t: number) => number;
  amount?: number;        // peak blur radius (default 5)
  opacity?: number;       // starting opacity (default 0)
}): TransitionConfig;
```

```svelte
<div transition:blur={{ amount: 10, duration: 400 }}>blurs in</div>
```

---

### `fly`

Translate from offset position while fading in.

```ts
fly(node: Element, params?: {
  delay?: number;
  duration?: number;
  easing?: (t: number) => number;
  x?: number;             // horizontal offset px (default 0)
  y?: number;             // vertical offset px (default 0)
  opacity?: number;       // starting opacity (default 0)
}): TransitionConfig;
```

```svelte
<div transition:fly={{ y: 200, duration: 300 }}>flies up from below</div>
```

Negative `x`/`y` flies in from the opposite direction.

---

### `slide`

Animate the element's `height` (or `width` on `axis: 'x'`).

```ts
slide(node: Element, params?: {
  delay?: number;
  duration?: number;
  easing?: (t: number) => number;
  axis?: 'x' | 'y';       // default 'y' (animates height)
}): TransitionConfig;
```

```svelte
<div transition:slide={{ axis: 'y', duration: 250 }}>expands</div>
<div transition:slide={{ axis: 'x' }}>slides horizontally</div>
```

---

### `scale`

Animate `transform: scale()` plus opacity.

```ts
scale(node: Element, params?: {
  delay?: number;
  duration?: number;
  easing?: (t: number) => number;
  start?: number;         // starting scale (default 0)
  opacity?: number;       // starting opacity (default 0)
}): TransitionConfig;
```

```svelte
<div transition:scale={{ start: 0.5, duration: 300 }}>scales up</div>
```

---

### `draw`

Animate `stroke-dashoffset` to "draw" an SVG path. The element must have a measurable `stroke` (typically a `<path>`).

```ts
draw(node: SVGElement, params?: {
  delay?: number;
  duration?: number | ((node: SVGElement, { from, to }) => number);
  easing?: (t: number) => number;
  speed?: number;          // alternative to duration (default 1)
}): TransitionConfig;
```

```svelte
<svg>
  <path
    transition:draw={{ duration: 1500 }}
    d="M10,80 Q52.5,10 95,80 T180,80"
    stroke="currentColor"
    fill="none"
    stroke-width="2"
  />
</svg>
```

---

### `crossfade`

Returns a paired `[send, receive]` for moving elements between keyed each blocks.

```ts
crossfade(params?: {
  delay?: number;
  duration?: number | ((node: Element, { from, to }) => number);
  easing?: (t: number) => number;
  fallback?: TransitionFn;   // used when no matching pair exists
}): [send: TransitionFn, receive: TransitionFn];
```

```svelte
<script>
  import { crossfade } from 'svelte/transition';
  const [send, receive] = crossfade({ duration: 300 });
</script>

{#each items as item (item.id)}
  <div in:receive={{ key: item.id }} out:send={{ key: item.id }}>
    {item.text}
  </div>
{/each}
```

Same `key` value pairs a `send` and `receive`. `fallback` runs when the pair cannot be matched (e.g. element added without a matching outgoing element).

---

## Transition Events

An element with transitions dispatches these in addition to standard DOM events:

| Event | Fires |
|-------|-------|
| `introstart` | Before the intro animation begins |
| `introend` | After the intro animation completes |
| `outrostart` | Before the outro animation begins |
| `outroend` | After the outro animation completes |

```svelte
<div
  transition:fly={{ y: 200, duration: 2000 }}
  onintrostart={() => (status = 'intro started')}
  onintroend={() => (status = 'intro ended')}
  onoutrostart={() => (status = 'outro started')}
  onoutroend={() => (status = 'outro ended')}
>
  Flies in and out
</div>
```

These bubble like normal DOM events.

---

## Local vs Global

By default, transitions are **local**: they only play when the *immediate* owning block is created or destroyed, **not** when an ancestor block is created or destroyed.

```svelte
{#if x}
  {#if y}
    <p transition:fade>plays only when y changes</p>
    <p transition:fade|global>plays when x OR y changes</p>
  {/if}
{/if}
```

Use `|global` when an element should animate in response to any ancestor block change.

---

## Block Outro Coordination

When a block (e.g. `{#if ...}`) is outroing, all elements **without their own transitions** are kept in the DOM until every outro in the block has completed. This prevents visual tearing.

```svelte
{#if show}
  <div class="card">
    <h1>Title</h1>
    <p transition:slide>Body slides out</p>
    <img src="/banner.png" />  <!-- waits for slide -->
  </div>
{/if}
```

---

## Custom Transition Function

```ts
// @noErrors
transition = (node: HTMLElement, params: any, options: { direction: 'in' | 'out' | 'both' }) => {
  delay?: number,
  duration?: number,
  easing?: (t: number) => number,
  css?: (t: number, u: number) => string,
  tick?: (t: number, u: number) => void
}
```

The third argument `options` contains `{ direction: 'in' | 'out' | 'both' }` — useful when the same function is used both as `in:` and `out:`.

### CSS-based (preferred)

```svelte
<script>
  import { elasticOut } from 'svelte/easing';

  function whoosh(node, { duration = 400 }) {
    const existing = getComputedStyle(node).transform.replace('none', '');
    return {
      duration,
      easing: elasticOut,
      css: (t) => `transform: ${existing} scale(${t});`
    };
  }
</script>

<div in:whoosh>whoosh</div>
```

### Tick-based

```svelte
<script>
  function typewriter(node, { speed = 1 }) {
    const text = node.textContent;
    return {
      duration: text.length / (speed * 0.01),
      tick: (t) => {
        node.textContent = text.slice(0, ~~(text.length * t));
      }
    };
  }
</script>

<p in:typewriter={{ speed: 1 }}>The quick brown fox</p>
```

### Function return (deferred)

If the transition returns a **function** instead of a config object, the function is called in the next microtask. This enables coordination between paired transitions (used by `crossfade`):

```js
function deferred(node, params) {
  return () => {
    // runs in next microtask — Svelte may have paired this with another
    return {
      duration: 300,
      css: (t) => `opacity: ${t}`
    };
  };
}
```

---

## Choosing `css` vs `tick`

| Use `css` when | Use `tick` when |
|----------------|-----------------|
| Animation can be expressed as CSS properties | You must mutate DOM that isn't a CSS property (e.g. `textContent`, scroll position) |
| Performance matters (runs off main thread) | Multiple coordinated changes per frame needed |
| You want Web Animations API features (e.g. easing) | Animation cannot be expressed as a transition property |

> [!NOTE] If it's possible to use `css` instead of `tick`, do so — web animations can run off the main thread, preventing jank on slower devices.

---

## Transition Lifecycle Summary

```
intro begins  →  introstart event
                ↓
            css/tick called repeatedly with t: 0 → 1
                ↓
            introend event
[ element visible ]
            ... (state change) ...
                ↓
            outrostart event
                ↓
            css/tick called repeatedly with t: 1 → 0
                ↓
            outroend event
[ element removed from DOM ]
```

---

## Related

- [examples/transition.md](../examples/transition.md) — basic built-in transition examples
- [examples/transitions-advanced.md](../examples/transitions-advanced.md) — crossfade, deferred, custom css/tick
- [examples/in-out-animate.md](../examples/in-out-animate.md) — `in:`/`out:` and `animate:`
- [references/in-out-reference.md](./in-out-reference.md) — `in:` / `out:` separate transitions