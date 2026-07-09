# Svelte Runtime References

In-depth technical references for Svelte 5's runtime APIs, covering mount/unmount, hydration, SSR, and component lifecycle internals.

## References Overview

| Reference | Description |
|-----------|-------------|
| [mount-unmount-reference.md](./mount-unmount-reference.md) | Deep dive into mount(), unmount(), hydrate() APIs and lifecycle |
| [ssr-reference.md](./ssr-reference.md) | Server-side rendering internals, async SSR, and CSP |

## Quick Reference

```js
// Mount options
mount(Component, {
  target: Element,        // Required: DOM target
  props?: {},             // Component props
  events?: {}             // Event handlers
});

// Unmount options
unmount(app, {
  outro?: boolean         // Wait for transitions (default: false)
});

// Hydrate options
hydrate(Component, {
  target: Element,        // Required: DOM target with SSR HTML
  props?: {},             // Component props
  events?: {}             // Event handlers
});

// Render options (SSR only)
render(Component, {
  props?: {},             // Component props
  csp?: { nonce?: string, hash?: boolean }
});

// Utility
flushSync();              // Force sync update
await tick();             // Wait for DOM update
```

## Navigation

- **Mount/Unmount/Hydrate**: [mount-unmount-reference.md](./mount-unmount-reference.md)
- **SSR/Render**: [ssr-reference.md](./ssr-reference.md)
