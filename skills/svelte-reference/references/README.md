# Svelte Reference Documentation

Complete API reference for Svelte's reactivity system including stores, motion, transitions, actions, and animations.

## Table of Contents

- [Store Reference](./store-reference.md) - writable, readable, derived, get, readonly
- [Motion Reference](./motion-reference.md) - tweened, spring, easing functions
- [Transition Reference](./transition-reference.md) - fade, fly, slide, scale, blur, draw
- [Transitions Complete](./transitions-complete.md) - Complete built-in transition API, events, |global, custom functions
- [in:/out: Reference](./in-out-reference.md) - Separate (unidirectional) in:/out: transitions
- [Animations Reference](./animations-reference.md) - animate: directive, flip, custom animation functions
- [use: Action Reference](./use-action-reference.md) - Action<>, ActionReturn, lifecycle, custom events

## Quick Import

```js
import { writable, readable, derived, get, readonly } from 'svelte/store';
import { tweened, spring } from 'svelte/motion';
import { fade, fly, slide, scale, blur, draw, crossfade } from 'svelte/transition';
import { flip } from 'svelte/animate';
import { cubicIn, cubicOut, elasticOut, bounceOut } from 'svelte/easing';
import { on } from 'svelte/events';
import type { Action } from 'svelte/action';
```

See [examples/README.md](../examples/README.md) for practical usage patterns.
