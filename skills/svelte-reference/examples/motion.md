# Motion Examples

## 1. tweened() Basic with Duration

```svelte
<script>
  import { tweened } from 'svelte/motion';
  import { cubicOut } from 'svelte/easing';

  const progress = tweened(0, {
    duration: 400
  });

  function handleClick() {
    $progress = Math.random();
  }
</script>

<button on:click={handleClick}>
  Animate
</button>

<progress value={$progress}></progress>
<p>{Math.round($progress * 100)}%</p>
```

---

## 2. tweened() with Easing

```svelte
<script>
  import { tweened } from 'svelte/motion';
  import { elasticOut, bounceOut } from 'svelte/easing';

  const position = tweened(0, {
    duration: 800,
    easing: elasticOut
  });

  const bounce = tweened(0, {
    duration: 600,
    easing: bounceOut
  });
</script>

<div>
  <button on:click={() => $position = 500}>
    Elastic: {$position.toFixed(0)}px
  </button>

  <button on:click={() => $bounce = 300}>
    Bounce: {$bounce.toFixed(0)}px
  </button>
</div>

<div style="transform: translateX({$position}px)">
  Elastic
</div>
```

---

## 3. tweened() with Custom Interpolator

```svelte
<script>
  import { tweened } from 'svelte/motion';

  // Custom color interpolator
  function interpolateColor(from, to, t) {
    const r = from.r + (to.r - from.r) * t;
    const g = from.g + (to.g - from.g) * t;
    const b = from.b + (to.b - from.b) * t;
    return { r: Math.round(r), g: Math.round(g), b: Math.round(b) };
  }

  const color = tweened(
    { r: 255, g: 0, b: 0 },
    {
      duration: 1000,
      interpolate: interpolateColor
    }
  );

  function randomColor() {
    color.set({
      r: Math.random() * 255,
      g: Math.random() * 255,
      b: Math.random() * 255
    });
  }
</script>

<button on:click={randomColor}>
  Change Color
</button>

<div
  style="background: rgb({$color.r}, {$color.g}, {$color.b})"
>
  RGB({$color.r.toFixed(0)}, {$color.g.toFixed(0)}, {$color.b.toFixed(0)})
</div>
```

---

## 4. spring() Basic

```svelte
<script>
  import { spring } from 'svelte/motion';

  const coords = spring({ x: 50, y: 50 }, {
    stiffness: 0.1,
    damping: 0.25
  });

  function handleMouseMove(e) {
    coords.set({ x: e.clientX, y: e.clientY });
  }
</script>

<svelte:window on:mousemove={handleMouseMove} />

<div
  style="transform: translate({$coords.x}px, {$coords.y}px)"
>
  Following mouse
</div>
```

---

## 5. spring() with Stiffness/Damping

```snip>
```svelte
<script>
  import { spring } from 'svelte/motion';

  // Tight spring - snappy feel
  const tight = spring(0, {
    stiffness: 0.8,
    damping: 0.3
  });

  // Loose spring - lazy, bouncy feel
  const loose = spring(0, {
    stiffness: 0.1,
    damping: 0.05
  });

  // Critically damped - smooth approach
  const smooth = spring(0, {
    stiffness: 0.3,
    damping: 0.8
  });
</script>

<div>
  <button on:click={() => { $tight = 100; $loose = 100; $smooth = 100; }}>
    Trigger All
  </button>
</div>

<div style="transform: translateX({$tight}px)">Tight (stiff: 0.8, damp: 0.3)</div>
<div style="transform: translateX({$loose}px)">Loose (stiff: 0.1, damp: 0.05)</div>
<div style="transform: translateX({$smooth}px)">Smooth (stiff: 0.3, damp: 0.8)</div>
```

**Stiffness/Damping Guide:**

| Stiffness | Damping | Feel |
|-----------|---------|------|
| High (0.8+) | Low | Snappy, oscillates |
| Low | Low | Bouncy, long oscillation |
| Any | High (0.7+) | Smooth, no overshoot |
| ~0.3 | ~0.8 | Balanced, natural |

---

## 6. spring() Value Changes

```svelte
<script>
  import { spring } from 'svelte/motion';

  const scale = spring(1, {
    stiffness: 0.2,
    damping: 0.4
  });

  let hovered = false;

  function handleMouseEnter() {
    hovered = true;
    $scale = 1.2;
  }

  function handleMouseLeave() {
    hovered = false;
    $scale = 1;
  }
</script>

<div
  on:mouseenter={handleMouseEnter}
  on:mouseleave={handleMouseLeave}
  style="transform: scale({$scale})"
>
  Hover me
</div>
```

**Set with velocity (no jump):**

```svelte
<script>
  import { spring } from 'svelte/motion';

  const pos = spring({ x: 0, y: 0 });

  // Update without jump even if current value differs
  pos.set({ x: 100, y: 100 }, { hard: true }); // Instant
  pos.set({ x: 200, y: 200 }, { soft: true }); // Smooth from current
</script>
```

---

## 7. Using Motion in $effect

```svelte
<script>
  import { tweened } from 'svelte/motion';
  import { cubicInOut } from 'svelte/easing';

  let userScore = 0;

  const displayScore = tweened(0, {
    duration: 500,
    easing: cubicInOut
  });

  // Automatically update tweened value when source changes
  $effect(() => {
    $displayScore = userScore;
  });
</script>

<button on:click={() => userScore += 100}>
  Add Score
</button>

<p>Display: {$displayScore.toFixed(0)}</p>
<p>Actual: {userScore}</p>
```

---

## 8. Motion for UI Animations

**Progress bar with tweened:**

```svelte
<script>
  import { tweened } from 'svelte/motion';
  import { quintOut } from 'svelte/easing';

  const progress = tweened(0, { duration: 400, easing: quintOut });

  let loading = false;

  async function simulateLoad() {
    loading = true;
    $progress = 0;

    for (let i = 1; i <= 100; i += 10) {
      await new Promise(r => setTimeout(r, 100));
      $progress = i / 100;
    }

    loading = false;
  }
</script>

<button on:click={simulateLoad} disabled={loading}>
  {loading ? 'Loading...' : 'Start'}
</button>

<progress value={$progress}></progress>
<div>{($progress * 100).toFixed(0)}%</div>
```

**Smoothed scroll indicator:**

```svelte
<script>
  import { tweened } from 'svelte/motion';
  import { cubicOut } from 'svelte/easing';

  const scrollY = tweened(0, {
    duration: 200,
    easing: cubicOut
  });

  function handleScroll() {
    $scrollY = window.scrollY;
  }
</script>

<svelte:window on:scroll={handleScroll} />

<div class="scroll-indicator" style="top: {$scrollY}px">
  Position: {$scrollY.toFixed(0)}px
</div>
```

---

## Summary

| Function | Use Case |
|----------|----------|
| `tweened(initial, { duration, easing })` | Linear interpolation over fixed time |
| `spring(initial, { stiffness, damping })` | Physics-based, responds to ongoing changes |
| `tweened` + custom `interpolate` | Colors, complex objects |
| `spring.set(value, { hard, soft })` | Control transition behavior |

**Choose `tweened` when:** Duration is fixed, value changes are discrete.

**Choose `spring` when:** Value changes continuously, natural feel is important.
