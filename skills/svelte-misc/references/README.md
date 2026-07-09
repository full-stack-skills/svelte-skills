# Svelte Misc References

In-depth reference documentation for Svelte 5's miscellaneous features.

## Contents

### Dynamic Components Reference
- [Dynamic Components Reference](./dynamic-reference.md) - `svelte:element`, `{#key}`, and component lifecycle

### TypeScript Reference
- [TypeScript Basics Reference](./typescript-reference.md) - TypeScript integration, types, and generics
- [TypeScript Advanced Reference](./typescript-advanced.md) - `Component` type, generic components, wrapper components, `$state` typing, `svelte/elements`, DOM type augmentation, `svelte-check`, action types, bindable props, common errors

### Custom Elements Reference
- [Custom Elements Reference](./custom-elements.md) - `customElement` option surface, `$host()` rune, lifecycle, slots, full caveats list, `ElementInternals` form association, shadow root modes

### Best Practices Reference
- [Best Practices Reference](./best-practices.md) - Comprehensive guide to `$state` / `$derived` / `$effect` patterns, props as changing values, `$inspect.trace`, events, snippets, keyed each blocks, CSS variables and child styling, context vs module state, async Svelte, testing notes, quick checklist

### Browser Support Reference
- [Browser Support Reference](./browser-support.md) - Minimum browser versions, feature exceptions, polyfills, SSR browser-API safety, bundler configuration (Vite / TypeScript), feature detection patterns, Playwright testing

## Topics Covered

- `svelte:element` vs `svelte:component`
- `{#key}` block behavior and use cases
- `{@html}` security model
- `{@debug}` triggering conditions
- `{@const}` usage patterns
- TypeScript type inference
- Generic components
- Event types
- Component types
- `svelte/elements` augmentation
- `Component` and `ComponentProps` types
- `svelte-check` in CI
- `customElement` compilation option
- Web Component lifecycle and slot semantics
- Shadow root modes (`open`, `closed`, `none`)
- `ElementInternals` form association
- `$state.raw` vs `$state`
- `$derived` vs `$effect`
- `$inspect.trace` for debugging
- Keyed each blocks for performance
- CSS variables for child component styling
- `createContext` for shared state
- Async / await in components
- Replacing legacy features
- Baseline 2020 browser matrix
- Feature-specific browser exceptions
- Polyfilling Web Components

## Quick Links

| Feature | Reference |
|---------|-----------|
| `svelte:element` | [Dynamic Components](./dynamic-reference.md#svelteelement) |
| `{#key}` | [Dynamic Components](./dynamic-reference.md#key-block) |
| `{@html}` | [Dynamic Components](./dynamic-reference.md#html) |
| `{@debug}` | [Dynamic Components](./dynamic-reference.md#debug) |
| `{@const}` | [Dynamic Components](./dynamic-reference.md#const) |
| TypeScript basics | [TypeScript Basics Reference](./typescript-reference.md) |
| TypeScript advanced | [TypeScript Advanced Reference](./typescript-advanced.md) |
| Generic components | [TypeScript Advanced Reference](./typescript-advanced.md#generic-props) |
| Wrapper components | [TypeScript Advanced Reference](./typescript-advanced.md#typing-wrapper-components) |
| `Component` type | [TypeScript Advanced Reference](./typescript-advanced.md#the-component-type) |
| `svelte/elements` augmentation | [TypeScript Advanced Reference](./typescript-advanced.md#svelteelements--enhancing-built-in-dom-types) |
| `customElement` options | [Custom Elements Reference](./custom-elements.md#customelement-option-surface) |
| `$host()` rune | [Custom Elements Reference](./custom-elements.md#host-rune) |
| Web Component lifecycle | [Custom Elements Reference](./custom-elements.md#component-lifecycle) |
| ElementInternals | [Custom Elements Reference](./custom-elements.md#elementinternals-form-association) |
| `$state` patterns | [Best Practices Reference](./best-practices.md#state) |
| `$derived` patterns | [Best Practices Reference](./best-practices.md#derived) |
| `$effect` patterns | [Best Practices Reference](./best-practices.md#effect) |
| Events | [Best Practices Reference](./best-practices.md#events) |
| Snippets | [Best Practices Reference](./best-practices.md#snippets) |
| Each blocks | [Best Practices Reference](./best-practices.md#each-blocks) |
| CSS variables | [Best Practices Reference](./best-practices.md#css-variables-from-js) |
| Context | [Best Practices Reference](./best-practices.md#context) |
| Browser matrix | [Browser Support Reference](./browser-support.md#minimum-browser-versions) |
| Feature exceptions | [Browser Support Reference](./browser-support.md#feature-specific-exceptions) |
| Polyfills | [Browser Support Reference](./browser-support.md#polyfills-for-older-browsers) |
| SSR safety | [Browser Support Reference](./browser-support.md#ssr) |
