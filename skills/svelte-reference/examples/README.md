# Svelte Reference Examples

Practical examples demonstrating Svelte's reactivity APIs including stores, motion, transitions, and animations.

## Table of Contents

- [Store Patterns](./store-patterns.md) - Writable, readable, derived stores, and subscription patterns
- [Motion](./motion.md) - Tweened and spring animations for UI effects
- [Transition](./transition.md) - Built-in and custom transition functions

## Quick Reference

```svelte
<script>
  import { writable, derived } from 'svelte/store';
  import { fade } from 'svelte/transition';
  import { tweened } from 'svelte/motion';
  import { cubicOut } from 'svelte/easing';
</script>
```

See [references/README.md](../references/README.md) for full API documentation.
