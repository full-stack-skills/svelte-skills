# Svelte Runtime Examples

Practical examples demonstrating Svelte 5's runtime APIs for mount/unmount, hydration, SSR, and component lifecycle management.

## Examples Overview

| Example | Description |
|---------|-------------|
| [mount-hydrate.md](./mount-hydrate.md) | Mount, unmount, hydrate, and render APIs with 9 practical examples |
| [ssr-patterns.md](./ssr-patterns.md) | Server-side rendering patterns including streaming, async SSR, and hydration |

## Quick Reference

```js
import { mount, unmount, hydrate, render, flushSync, tick } from 'svelte';

// Client-side mounting
const app = mount(App, { target: document.body, props: { name: 'world' } });

// Unmount with outro animation
unmount(app, { outro: true });

// Hydrate SSR HTML
const app = hydrate(App, { target: document.body });

// Server-side render
const { head, body } = await render(App, { props: { data } });

// Force sync updates in tests
flushSync();

// Wait for DOM update
await tick();
```

## Navigation

- **Getting Started**: [mount-hydrate.md](./mount-hydrate.md)
- **SSR Patterns**: [ssr-patterns.md](./ssr-patterns.md)
