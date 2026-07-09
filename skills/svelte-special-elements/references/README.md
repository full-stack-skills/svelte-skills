# Svelte Special Elements Reference

Detailed reference documentation for all `svelte:` special elements and compiler options.

## References

| File | Description |
|------|-------------|
| [boundary-reference.md](./boundary-reference.md) | `<svelte:boundary>` — `pending` / `failed` / `onerror` properties, `transformError` (5.51+), nesting, what is/isn't caught, common pitfalls |
| [svelte-head-reference.md](./svelte-head-reference.md) | `<svelte:head>` — supported elements, SSR head extraction, hydration, reactivity, constraints |
| [svelte-window-document-body-reference.md](./svelte-window-document-body-reference.md) | All events and bindable properties for `<svelte:window>` / `<svelte:document>` / `<svelte:body>` |
| [svelte-element-reference.md](./svelte-element-reference.md) | `<svelte:element>`, `<svelte:component>`, `<svelte:self>`, `<svelte:fragment>` |
| [svelte-options-reference.md](./svelte-options-reference.md) | `<svelte:options>` — `customElement`, `runes`, `namespace`, `css`, and all compiler options |

## Quick Reference

| Element | Purpose | Added in |
|---------|---------|----------|
| `svelte:boundary` | Error and async boundaries (rendering errors + initial `await` pending) | 5.3.0 |
| `svelte:window` | Window events and properties | — |
| `svelte:document` | Document-level events | — |
| `svelte:body` | Body-level mouse events | — |
| `svelte:head` | Document head content (separately exposed during SSR) | — |
| `svelte:element` | Dynamic HTML tag rendering | — |
| `svelte:component` | Dynamic component rendering (Svelte 4 style) | — |
| `svelte:options` | Compiler configuration | — |
| `svelte:fragment` | Slot fallback content | — |
| `svelte:self` | Recursive component (legacy Svelte 4) | — |
