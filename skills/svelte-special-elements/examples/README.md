# Svelte Special Elements Examples

Practical examples demonstrating all `svelte:` special elements and their common use cases.

## Examples

| File | Description |
|------|-------------|
| [svelte-element-examples.md](./svelte-element-examples.md) | Comprehensive examples covering `svelte:window`, `svelte:document`, `svelte:body`, `svelte:element`, `svelte:component`, `svelte:self`, `svelte:options` |

## Quick Navigation

### svelte:window
- Event listeners: `onkeydown`, `onresize`, `onscroll`
- Bindable properties: `bind:innerWidth`, `bind:innerHeight`, `bind:scrollX`, `bind:scrollY`

### svelte:document
- Events: `onvisibilitychange`, `onselectionchange`
- Readonly bindings: `visibilityState`, `activeElement`

### svelte:body
- Events: `onmouseenter`, `onmouseleave`

### svelte:element
- Dynamic tag switching (h1/h2/h3)
- Conditional rendering with if blocks
- SVG namespace handling

### svelte:component
- Svelte 4 style dynamic component (still works in Svelte 5)

### svelte:self
- Recursive component pattern (legacy Svelte 4 style)

### svelte:options
- `customElement` with `$host`
- `runes={false}` for legacy mode
- `namespace="svg"`
