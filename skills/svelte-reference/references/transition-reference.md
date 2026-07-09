# Transition API Reference

## Overview

```svelte
<script>
  import { fade, fly, slide, scale, blur, draw, crossfade } from 'svelte/transition';
  import { cubicOut } from 'svelte/easing';
</script>
```

---

## Built-in Transitions

### fade

Fades element by animating opacity.

```ts
fade(node: HTMLElement, {
  delay?: number,      // default: 0
  duration?: number,   // default: 400
  easing?: function    // default: linear
}) => TransitionConfig
```

**Example:**
```svelte
{#if show}
  <div transition:fade={{ duration: 300 }}>Fading</div>
{/if}
```

---

### fly

Moves element while fading. Animates from a position.

```ts
fly(node: HTMLElement, {
  delay?: number,      // default: 0
  duration?: number,   // default: 400
  easing?: function,   // default: linear
  x?: number,          // horizontal offset (default: 0)
  y?: number           // vertical offset (default: 200)
}) => TransitionConfig
```

**Example:**
```svelte
<div transition:fly={{ y: 50, duration: 300 }}>Flying</div>
```

---

### slide

Animates height of element.

```ts
slide(node: HTMLElement, {
  delay?: number,      // default: 0
  duration?: number,   // default: 400
  easing?: function,   // default: linear
  axis?: 'x' | 'y'     // default: 'y'
}) => TransitionConfig
```

**Example:**
```svelte
{#if expanded}
  <div transition:slide>Content</div>
{/if}
```

---

### blur

Applies blur effect while fading.

```ts
blur(node: HTMLElement, {
  delay?: number,      // default: 0
  duration?: number,   // default: 400
  easing?: function,   // default: linear
  amount?: number      // blur amount in pixels (default: 5)
}) => TransitionConfig
```

**Example:**
```svelte
<div transition:blur={{ amount: 10 }}>Blurring</div>
```

---

### scale

Scales element while fading.

```ts
scale(node: HTMLElement, {
  delay?: number,      // default: 0
  duration?: number,   // default: 400
  easing?: function,   // default: linear
  start?: number,      // starting scale (default: 0)
  opacity?: number     // starting opacity (default: 0)
}) => TransitionConfig
```

**Example:**
```svelte
<div transition:scale={{ start: 0.5 }}>Scaling</div>
```

---

### draw

Animates SVG path drawing.

```ts
draw(node: HTMLElement, {
  delay?: number,      // default: 0
  duration?: number,   // default: 400
  easing?: function    // default: linear
}) => TransitionConfig
```

**Example:**
```svelte
<svg>
  <path transition:draw d="M0,0 L100,100" />
</svg>
```

---

### crossfade

Creates paired transitions between elements.

```ts
crossfade({
  duration?: number,   // default: 400
  delay?: number,      // default: 0
  easing?: function,   // default: linear
  fallback?: function  // fallback transition for non-paired
}) => [send: TransitionFn, receive: TransitionFn]
```

**Example:**
```svelte
<script>
  import { crossfade } from 'svelte/transition';

  const [send, receive] = crossfade({ duration: 300 });
</script>

{#if selected}
  <div in:receive={{ key: item.id }} out:send={{ key: item.id }}>
    {item.name}
  </div>
{/if}
```

---

## Transition Parameters

All transitions accept this common parameters object:

```ts
{
  delay?: number,       // ms before starting (default: 0)
  duration?: number,    // ms duration (default: 400)
  easing?: function,    // (t: number) => number (default: linear)
  css?: (t: number, u: number) => string,  // custom CSS generator
  tick?: (t: number, u: number) => void    // custom JS callback
}
```

**Note:** `css` and `tick` are mutually exclusive.

---

## In/Out Transitions

Use `in:` and `out:` for different enter/exit transitions.

```svelte
{#if show}
  <div in:fade={{ delay: 0 }} out:fly={{ y: -50 }}>
    Different enter/exit
  </div>
{/if}
```

**Common patterns:**

```svelte
<!-- Fade in, fly out -->
<div in:fade out:fly={{ y: 100 }}>

<!-- Scale in, fade out -->
<div in:scale={{ start: 0.8 }} out:fade>

<!-- Staggered transitions -->
<div in:fade={{ delay: 100 }} out:fade={{ delay: 0 }}>
```

---

## Custom Transition Factories

Return a `TransitionConfig` object from a function:

```ts
function customTransition(
  node: HTMLElement,
  { speed = 50, ... }
) {
  return {
    delay?: number,
    duration?: number,
    easing?: function,
    css?: (t: number, u: number) => string,
    tick?: (t: number, u: number) => void
  };
}
```

**CSS-based custom transition:**
```svelte
<script>
  function glow(node, { duration = 400, color = '#ff0000' }) {
    return {
      duration,
      css: (t) => `
        box-shadow: 0 0 ${10 + t * 20}px ${color};
      `
    };
  }
</script>

<div transition:glow={{ color: '#00ff00' }}>Glowing</div>
```

**Tick-based custom transition:**
```svelte
<script>
  function typewriter(node, { speed = 50 }) {
    const text = node.textContent;
    return {
      duration: text.length * speed,
      tick: (t) => {
        node.textContent = text.slice(0, Math.floor(text.length * t));
      }
    };
  }
</script>

<div transition:typewriter>Hello, World!</div>
```

---

## Transition Events

Transitions dispatch DOM events:

```svelte
<div
  transition:fade={{ duration: 300 }}
  on:introstart={() => console.log('intro starts')}
  on:introend={() => console.log('intro ends')}
  on:outrostart={() => console.log('outro starts')}
  on:outroend={() => console.log('outro ends')}
>
  Content
</div>
```

**Event lifecycle:**
1. `introstart` - Before enter animation begins
2. `introend` - After enter animation completes
3. `outrostart` - Before exit animation begins
4. `outroend` - After exit animation completes

---

## local vs global Transitions

By default, transitions are `local` - they only trigger when their immediate condition changes.

**Global modifier:**
```svelte
<!-- Triggers even if parent condition changed -->
<div transition:fade|global>
  Global transition
</div>

<!-- Local only (default) -->
<div transition:fade>
  Local transition
</div>
```

**Use `|global` when:**
- Element should animate even when parent control flow changes
- Cross-component coordinated transitions
