# Motion API Reference

## Overview

```js
import { tweened, spring } from 'svelte/motion';
import { cubicIn, cubicOut, elasticOut, bounceOut } from 'svelte/easing';
```

---

## tweened(initialValue, options?)

Creates a store that smoothly transitions to new values over a fixed duration.

**Signature:**
```ts
function tweened<T>(
  initialValue: T,
  options?: {
    duration?: number;
    easing?: (t: number) => number;
    interpolate?: (from: T, to: T, t: number) => T;
  }
): {
  subscribe: (run: (value: T) => void) => () => void;
  set: (value: T, options?: { duration?, easing?, interpolate? }) => void;
}
```

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `duration` | `number` | `400` | Transition time in ms |
| `easing` | `function` | `linear` | Easing function (0-1 to 0-1) |
| `interpolate` | `function` | - | Custom interpolation function |

**Basic example:**
```js
const progress = tweened(0, { duration: 300 });
$progress = 1; // Animates from 0 to 1 over 300ms
```

**With easing:**
```js
import { cubicOut, elasticOut } from 'svelte/easing';

const smooth = tweened(0, {
  duration: 800,
  easing: cubicOut
});

const bouncy = tweened(0, {
  duration: 600,
  easing: elasticOut
});
```

---

## spring(initialValue, options?)

Creates a store that smoothly transitions using physics-based motion.

**Signature:**
```ts
function spring<T>(
  initialValue: T,
  options?: {
    stiffness?: number;   // 0-1, default 0.1
    damping?: number;     // 0-1, default 0.4
    precision?: number;   // default 0.01
  }
): {
  subscribe: (run: (value: T) => void) => () => void;
  set: (value: T, options?: { hard?, soft? }) => void;
  update: (fn: (value: T) => T, options?: { hard?, soft? }) => void;
}
```

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `stiffness` | `number` | `0.1` | Spring stiffness (0-1) |
| `damping` | `number` | `0.4` | Damping ratio (0-1) |
| `precision` | `number` | `0.01` | Target threshold for stopping |

**Basic example:**
```js
const coords = spring({ x: 0, y: 0 }, {
  stiffness: 0.2,
  damping: 0.5
});

$coords = { x: 100, y: 200 }; // Springs to new position
```

**Set options:**

| Option | Effect |
|--------|--------|
| `hard: true` | Instant transition (no animation) |
| `soft: true | number` | Smooth from current position |

---

## Standard Easings

Import from `svelte/easing`:

```js
import {
  linear,
  cubicIn, cubicOut, cubicInOut,
  quadIn, quadOut, quadInOut,
  quartIn, quartOut, quartInOut,
  quintIn, quintOut, quintInOut,
  sineIn, sineOut, sineInOut,
  expoIn, expoOut, expoInOut,
  circIn, circOut, circInOut,
  elasticIn, elasticOut, elasticInOut,
  backIn, backOut, backInOut,
  bounceIn, bounceOut, bounceInOut
} from 'svelte/easing';
```

**Easing categories:**

| Category | Functions | Feel |
|----------|-----------|------|
| Linear | `linear` | Constant speed |
| Quadratic | `quadIn/Out/InOut` | Moderate acceleration |
| Cubic | `cubicIn/Out/InOut` | Standard easing |
| Quart/Quint | `*In/Out/InOut` | More pronounced |
| Sinusoidal | `sineIn/Out/InOut` | Smooth |
| Exponential | `expo*` | Very pronounced |
| Circular | `circ*` | Fast then slow |
| Elastic | `elastic*` | Bouncy overshoot |
| Back | `back*` | Slight overshoot |
| Bounce | `bounce*` | Physical bounce |

**Usage:**
```svelte
<div transition:fly={{ y: 100, easing: cubicOut }}>
  Smooth fly in
</div>
```

---

## Custom interpolate Function

For non-numeric values like colors or objects.

**Signature:**
```ts
interpolate: (from: T, to: T, t: number) => T
// t goes from 0 to 1
```

**Color interpolation:**
```js
import { tweened } from 'svelte/motion';

function interpolateColor(from, to, t) {
  return {
    r: Math.round(from.r + (to.r - from.r) * t),
    g: Math.round(from.g + (to.g - from.g) * t),
    b: Math.round(from.b + (to.b - from.b) * t)
  };
}

const color = tweened(
  { r: 255, g: 0, b: 0 },
  { interpolate: interpolateColor }
);

color.set({ r: 0, g: 0, b: 255 }); // Animates red to blue
```

---

## When to Use tweened vs spring

| Scenario | Use |
|----------|-----|
| Fixed-duration animation | `tweened` |
| Progress indicators | `tweened` |
| User interactions that update continuously | `spring` |
| Following mouse/touch | `spring` |
| Physics-like motion | `spring` |
| Color transitions | `tweened` with custom `interpolate` |
| Size transitions | Either |

**tweened:** Value changes are discrete, you know the start and end.

**spring:** Value changes are ongoing, you need natural feel.

---

## Stiffness and Damping Guide

| Stiffness | Damping | Behavior |
|-----------|---------|----------|
| 0.1 | 0.4 | Default, balanced |
| 0.8+ | 0.3- | Snappy, may oscillate |
| 0.1 | 0.05 | Very bouncy, slow to settle |
| 0.3 | 0.8+ | Smooth, no overshoot |

**Formula approximation:**
- **Critical damping:** `damping ≈ 2 * sqrt(stiffness)`
- **Over-damped:** `damping > 2 * sqrt(stiffness)`
- **Under-damped:** `damping < 2 * sqrt(stiffness)` (oscillation)
