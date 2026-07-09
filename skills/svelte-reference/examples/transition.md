# Transition Examples

## 1. fade Transition

```svelte
<script>
  let visible = true;
</script>

<button on:click={() => visible = !visible}>
  Toggle
</button>

{#if visible}
  <div transition:fade>
    Fades in and out
  </div>
{/if}
```

**With parameters:**

```svelte
<div transition:fade={{ duration: 500, delay: 100 }}>
  Slow fade
</div>
```

---

## 2. fly Transition

```svelte
<script>
  let visible = false;
</script>

<button on:click={() => visible = !visible}>
  Toggle
</button>

{#if visible}
  <div transition:fly={{ y: 200, duration: 300 }}>
    Flies in from below
  </div>
{/if}
```

**Different directions:**

```svelte
<!-- From top -->
<div transition:fly={{ y: -200 }}>Top</div>

<!-- From left -->
<div transition:fly={{ x: -200 }}>Left</div>

<!-- From right -->
<div transition:fly={{ x: 200 }}>Right</div>
```

---

## 3. slide Transition

```svelte
<script>
  let expanded = false;
</script>

<button on:click={() => expanded = !expanded}>
  {expanded ? 'Collapse' : 'Expand'}
</button>

{#if expanded}
  <div transition:slide={{ duration: 200 }}>
    <p>This content slides in and out</p>
    <p>Uses the height property</p>
  </div>
{/if}
```

**Slide with axis option:**

```svelte
<!-- Horizontal slide (uses width) -->
<div transition:slide={{ axis: 'x' }}>
  Slides horizontally
</div>
```

---

## 4. blur Transition

```svelte
<script>
  let visible = true;
</script>

<button on:click={() => visible = !visible}>
  Toggle
</button>

{#if visible}
  <div transition:blur={{ amount: 10, duration: 400 }}>
    Blurs in and out
  </div>
{/if}
```

---

## 5. scale Transition

```svelte
<script>
  let visible = true;
</script>

<button on:click={() => visible = !visible}>
  Toggle
</button>

{#if visible}
  <div transition:scale={{ start: 0.5, duration: 300 }}>
    Scales from 0.5 to 1
  </div>
{/if}
```

**With opacity:**

```svelte
<div transition:scale={{ start: 0, opacity: 1, duration: 300 }}>
  Starts invisible at scale 0
</div>
```

---

## 6. Custom Transition Function

```svelte
<script>
  function typewriter(node, { speed = 50, delay = 0 }) {
    const text = node.textContent;
    const duration = text.length * speed;

    return {
      delay,
      duration,
      tick: (t) => {
        const i = Math.floor(text.length * t);
        node.textContent = text.slice(0, i);
      }
    };
  }
</script>

<div transition:typewriter={{ speed: 30 }}>
  Custom typewriter effect
</div>
```

**Custom CSS transition:**

```svelte
<script>
  function glow(node, { duration = 400, color = '#ff0000' }) {
    return {
      duration,
      css: (t) => `
        box-shadow: 0 0 ${10 + t * 20}px ${color};
        opacity: ${0.5 + t * 0.5};
      `
    };
  }
</script>

<div transition:glow={{ color: '#00ff00' }}>
  Glowing box
</div>
```

---

## 7. Transition with Parameters

```svelte
<script>
  import { fly, fade } from 'svelte/transition';
  import { elasticOut } from 'svelte/easing';

  let show = false;

  const config = {
    y: 100,
    duration: 500,
    easing: elasticOut
  };
</script>

<button on:click={() => show = !show}>
  Toggle
</button>

{#if show}
  <!-- Static config -->
  <div transition:fly={config}>
    Configurable transition
  </div>

  <!-- Dynamic config -->
  <div transition:fly={{ y: 50, duration: 300, delay: 100 }}>
    Dynamic parameters
  </div>
{/if}
```

---

## 8. In/out Transitions

```svelte
<script>
  import { fade, fly } from 'svelte/transition';
</script>

{#if visible}
  <!-- Different transitions for enter/exit -->
  <div in:fade={{ duration: 200 }} out:fly={{ y: -50 }}>
    Fade in, fly out
  </div>

  <!-- Same transition, different params -->
  <div in:fade={{ delay: 0 }} out:fade={{ delay: 200 }}>
    Staggered fade
  </div>
{/if}
```

**Using crossfade for paired transitions:**

```svelte
<script>
  import { crossfade } from 'svelte/transition';

  const [send, receive] = crossfade({
    duration: 300,
    fallback: fade
  });

  let items = ['One', 'Two', 'Three'];
  let selected = null;
</script>

<div class="container">
  {#each items as item (item)}
    <div
      class:selected={selected === item}
      on:click={() => selected = selected === item ? null : item}
      in:receive={{ key: item }}
      out:send={{ key: item }}
    >
      {item}
    </div>
  {/each}
</div>
```

---

## Transition Parameters Reference

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `delay` | `number` | `0` | Delay before starting (ms) |
| `duration` | `number` | `400` | Duration of transition (ms) |
| `easing` | `function` | `linear` | Easing function |
| `css` | `function` | - | Custom CSS generator `(t, u) => string` |
| `tick` | `function` | - | Custom JS callback `(t, u) => void` |

**Note:** `css` and `tick` are mutually exclusive per transition.

---

## Summary

| Transition | Effect | Key Params |
|------------|--------|------------|
| `fade` | Opacity | `delay`, `duration` |
| `fly` | Position | `x`, `y`, `duration` |
| `slide` | Height | `axis`, `duration` |
| `blur` | Blur + opacity | `amount`, `duration` |
| `scale` | Scale + opacity | `start`, `duration` |
| `crossfade` | Paired send/receive | `duration`, `fallback` |

**Use `in:` and `out:` when enter and exit transitions differ.**
