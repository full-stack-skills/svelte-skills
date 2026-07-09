# Svelte Misc Examples

This directory contains practical examples for Svelte 5's miscellaneous features.

## Contents

### Dynamic Components
- [Dynamic Components](./dynamic-components.md) - `svelte:element` and `{#key}` block examples

### Debug and HTML
- [Debug and HTML](./debug-html.md) - `@html`, `@debug`, and `@const` examples

### TypeScript
- [TypeScript Basics](./typescript.md) - TypeScript integration with Svelte 5
- [TypeScript Advanced](./typescript-advanced.md) - Generics, wrapper components, `Component` type, `svelte/elements` augmentation, `svelte-check`, `$state` in classes, `$bindable`, form state

### Custom Elements
- [Custom Elements](./custom-elements.md) - 12 patterns for compiling Svelte components to Web Components: `customElement` option, `$host()`, attribute reflection, type coercion, `extend`, `ElementInternals`, slot behavior, `on*` caveat

### Best Practices
- [Best Practices](./best-practices.md) - 14 patterns: `$state.raw`, `$derived.by`, escaping `$effect`, treated-as-changing props, events, snippets, keyed each blocks, CSS variables, `createContext`, async Svelte, legacy replacements

### Migration
- [Svelte 3 → 4 Migration](./migration-v4.md) - 10 patterns: `tag` → `customElement`, stricter types, transition local-by-default, preprocessor order, CJS removal, minimum versions

### Browser Support
- [Browser Support](./browser-support.md) - 7 patterns: feature exceptions, polyfills, bundler target, SSR guards, `$state.snapshot` fallback, Playwright browser-version pinning

## Prerequisites

- Svelte 5 with runes enabled
- TypeScript 5+ (for TypeScript examples)

## Running Examples

All examples assume a standard Svelte 5 project setup:

```bash
npm create svelte@latest my-app
cd my-app
npm install
npm run dev
```
