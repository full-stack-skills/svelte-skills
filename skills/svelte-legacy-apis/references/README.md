# Svelte Legacy APIs - References

This directory contains complete reference documentation for Legacy APIs in Svelte 5.

## Contents

- **[migration-reference.md](./migration-reference.md)** - Complete migration reference from Svelte 4 to Svelte 5
- **[legacy-mode-reference.md](./legacy-mode-reference.md)** - Legacy mode behavior and usage guide
- **[store-reference.md](./store-reference.md)** - Complete store API reference

## Quick Reference Table

| Legacy (Svelte 4) | Modern (Svelte 5) |
|-------------------|-------------------|
| `let x = val` (top-level reactive) | `let x = $state(val)` |
| `$: x = expr` | `let x = $derived(expr)` |
| `$: { if (cond) x = v }` | `$effect(() => { if (cond) x = v })` |
| `export let prop` | `let { prop } = $props()` |
| `$$props` | `let { ...rest } = $props()` |
| `<slot />` | `{@render children()}` |
| `createEventDispatcher` | callback props |
| `on:click={fn}` | `onclick={fn}` |
| `<svelte:component this={x}>` | `<svelte:element this={x}>` |
| `<svelte:self>` | `import Self from './This.svelte'` |
| `class:cool={bool}` | `class={{ cool: bool }}` |
| `let:slotProp={value}` | Props in snippets |

## Navigation

For practical examples, see the [examples](../examples/README.md) directory.
