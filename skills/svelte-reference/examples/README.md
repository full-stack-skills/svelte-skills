# Svelte Reference Examples

Practical examples demonstrating Svelte's reactivity APIs including stores, motion, transitions, actions, and animations.

## Table of Contents

- [Store Patterns](./store-patterns.md) - Writable, readable, derived stores, and subscription patterns
- [Motion](./motion.md) - Tweened and spring animations for UI effects
- [Transition](./transition.md) - Built-in and custom transition functions
- [Transitions Advanced](./transitions-advanced.md) - Crossfade, deferred transitions, transition events, custom css/tick
- [in:/out:/animate:](./in-out-animate.md) - Separate enter/exit transitions, animate:flip, custom animation functions
- [use: Action](./use-action.md) - use: directive with typing, lifecycle, custom events

## Quick Reference

```svelte
<script>
  import { writable, derived } from 'svelte/store';
  import { fade, fly } from 'svelte/transition';
  import { flip } from 'svelte/animate';
  import { tweened } from 'svelte/motion';
  import { cubicOut } from 'svelte/easing';
</script>
```

See [references/README.md](../references/README.md) for full API documentation.
