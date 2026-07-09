# Svelte Special Elements Examples

Practical examples demonstrating all `svelte:` special elements and their common use cases.

## Examples

| File | Description |
|------|-------------|
| [boundary-examples.md](./boundary-examples.md) | `<svelte:boundary>` — `pending` / `failed` / `onerror`, error reporting, nested boundaries, SSR `transformError` |
| [svelte-head-examples.md](./svelte-head-examples.md) | `<svelte:head>` — SEO meta, Open Graph, Twitter Card, JSON-LD, dynamic stylesheets, SSR head extraction |
| [svelte-element-examples.md](./svelte-element-examples.md) | `<svelte:window>`, `<svelte:document>`, `<svelte:body>`, `<svelte:element>`, `<svelte:component>`, `<svelte:self>`, `<svelte:options>` |

## Quick Navigation

### svelte:boundary (5.3+)
- `pending` snippet for initial async loading
- `failed` snippet for error fallback with `reset`
- `onerror` handler for error reporting (Sentry, etc.)
- Nesting for granular recovery per region
- `transformError` for SSR (5.51+)

### svelte:window
- Event listeners: `onkeydown`, `onresize`, `onscroll`
- Bindable properties: `bind:innerWidth`, `bind:innerHeight`, `bind:scrollX`, `bind:scrollY`, `bind:online`, `bind:devicePixelRatio`

### svelte:document
- Events: `onvisibilitychange`, `onselectionchange`, `onreadystatechange`, `onfullscreenchange`
- Readonly bindings: `bind:visibilityState`, `bind:activeElement`, `bind:fullscreenElement`, `bind:pointerLockElement`

### svelte:body
- Events: `onmouseenter`, `onmouseleave` (do not fire on `window`)
- Actions: `use:trapFocus` and other body-level actions

### svelte:head
- `<title>` from reactive state
- SEO `<meta>` and Open Graph tags
- JSON-LD structured data
- Per-page stylesheet loading
- Canonical URL

### svelte:element
- Dynamic tag switching (h1/h2/h3)
- Conditional rendering with `{#if}` blocks
- SVG namespace handling with `xmlns`

### svelte:component
- Svelte 4 style dynamic component (still works in Svelte 5)

### svelte:self
- Recursive component pattern (legacy Svelte 4 style)

### svelte:options
- `customElement` with `$host`
- `runes={false}` for legacy mode
- `namespace="svg"`
