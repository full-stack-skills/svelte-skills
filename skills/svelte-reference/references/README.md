# Svelte Reference Documentation

Complete API reference for Svelte's reactivity system including stores, motion, transitions, and animations.

## Table of Contents

- [Store Reference](./store-reference.md) - writable, readable, derived, get
- [Motion Reference](./motion-reference.md) - tweened, spring, easing functions
- [Transition Reference](./transition-reference.md) - fade, fly, slide, custom transitions

## Quick Import

```js
import { writable, readable, derived, get } from 'svelte/store';
import { tweened, spring } from 'svelte/motion';
import { fade, fly, slide, scale } from 'svelte/transition';
import { cubicIn, cubicOut, elasticOut } from 'svelte/easing';
import { flip } from 'svelte/animate';
```

See [examples/README.md](../examples/README.md) for practical usage patterns.
