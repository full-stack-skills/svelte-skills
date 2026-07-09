# Svelte Runtime Examples

Practical examples demonstrating Svelte 5's runtime APIs for mount/unmount, hydration, SSR, lifecycle hooks, and CSP.

## Examples Overview

| Example | Description |
|---------|-------------|
| [mount-hydrate.md](./mount-hydrate.md) | Mount, unmount, hydrate, and render APIs with 9 practical examples |
| [ssr-patterns.md](./ssr-patterns.md) | Server-side rendering patterns including streaming, async SSR, and hydration |
| [onmount-tick.md](./onmount-tick.md) | 15 lifecycle examples: onMount (sync/async/cleanup/AbortController), onDestroy, tick for focus/measure, $effect.pre chat autoscroll |
| [csp-examples.md](./csp-examples.md) | 8 CSP examples: nonce for Express/SvelteKit/Cloudflare, hash for SSG, runtime configuration |

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
const { head, body, hashes } = await render(App, { props: { data }, csp: { nonce } });

// Force sync updates in tests
flushSync();

// Wait for DOM update
await tick();
```

## Navigation

- **Getting Started**: [mount-hydrate.md](./mount-hydrate.md)
- **SSR Patterns**: [ssr-patterns.md](./ssr-patterns.md)
- **Lifecycle Hooks**: [onmount-tick.md](./onmount-tick.md)
- **CSP**: [csp-examples.md](./csp-examples.md)
