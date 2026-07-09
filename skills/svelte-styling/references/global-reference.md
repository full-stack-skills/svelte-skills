# `:global(...)` Reference

A complete reference for the `:global(...)` modifier and the `:global { ... }` block — the two ways to opt a selector out of Svelte's scoping mechanism.

## Forms at a Glance

| Form | Use it for |
|------|-----------|
| `:global(selector)` | One selector (tag, class, attribute, compound) bypasses scoping |
| `:global(selector1 selector2)` | Compound selector bypasses scoping as a single unit |
| `:global { ... }` | A whole block of selectors is emitted without scope hash |
| `parent :global { ... }` | A scoped ancestor followed by an unscoped descendant block |
| `@keyframes -global-name` | Keyframe name emitted without scope hash |

## Single-Selector `:global(selector)`

### Tag selector

```svelte
<style>
  :global(body) { margin: 0; }
  :global(html) { height: 100%; }
</style>
```

The selector receives no scope hash. It matches the actual element anywhere in the document.

### Class selector

```svelte
<style>
  :global(.app-shell) { display: grid; min-height: 100vh; }
</style>
```

Matches any element with `class="app-shell"`, regardless of which component rendered it.

### Attribute selector

```svelte
<style>
  :global([data-theme="dark"]) { background: #111; color: #eee; }
</style>
```

Useful for opt-in theming applied to a parent element.

### ID selector

```svelte
<style>
  :global(#root) { isolation: isolate; }
</style>
```

IDs are already globally unique, so `:global(#root)` and `#root` (in global CSS) are equivalent — but the `:global(...)` form makes the intent clear inside a scoped block.

## Compound Selector

A *compound* selector (`a.b`, `a.b.c`, `tag.class`, etc.) inside `:global(...)` is treated as a single unit. The **element** still needs to belong to the component's scope (the compiler adds the scope class to it), but the trigger class(es) can be added externally — even programmatically.

```svelte
<style>
  p:global(.big.red) { font-weight: bold; font-size: 1.25rem; }
</style>

<p class="big red">Styled</p>
```

Compiled:

```css
p.svelte-xyz.big.red { font-weight: bold; font-size: 1.25rem; }
```

The `<p>` still gets the scope class, so the rule only matches `<p>` elements in this component. The `.big.red` part is not scoped — the rule triggers when an element in this component has those classes, whether the classes were put there by the template or by a library at runtime.

### Why this matters

If a third-party library adds a class to your element after the initial render, the rule will still apply — the class doesn't need to be present in your `.svelte` source.

## Nested Global (Scoped + Unscoped Combined)

The most common form: keep the **outer** part scoped (so the rule only matches inside your component), but make the **inner** part unscoped (so it reaches across component boundaries).

```svelte
<style>
  /* 'div' is scoped (this component's scope class added);
     'strong' is unscoped (matches any <strong>). */
  div :global(strong) {
    color: goldenrod;
  }
</style>

<div>
  <!-- This <strong> matches even if it's from a child component -->
  <strong>Golden text</strong>
</div>
```

Compiled:

```css
div.svelte-xyz strong { color: goldenrod; }
```

The `strong` is not given the scope class, so the rule can reach into nested components or `{@html}` content.

## The `:global { ... }` Block

For multiple selectors that all need to be global.

```svelte
<style>
  :global {
    /* Every div in the application */
    div { box-sizing: border-box; }

    /* Theme tokens */
    :root {
      --bg: #ffffff;
      --text: #111111;
    }

    /* Multiple selectors share the same rule */
    h1, h2, h3 { margin: 0; }
  }
</style>
```

All rules inside the block are emitted without scope hashes.

### Scoped Prefix + Unscoped Block

You can prefix the block with a scoped ancestor — the prefix is scoped, everything inside the block is unscoped.

```svelte
<style>
  /* .tooltip is scoped; the descendants are not. */
  .tooltip :global {
    .arrow { fill: currentColor; }
    .body  { background: black; color: white; }
  }
</style>
```

This is equivalent to:

```svelte
<style>
  .tooltip :global .arrow { fill: currentColor; }
  .tooltip :global .body  { background: black; color: white; }
</style>
```

The block form is preferred when you have several descendants to list.

> The Svelte docs note: "The second example above could also be written as an equivalent `.a :global .b .c .d` selector, where everything after the `:global` is unscoped, though the nested form is preferred." ([Svelte docs](https://svelte.dev/docs/svelte/global-styles))

## `:global` and the Cascade

`:global` does **not** lower specificity. If the surrounding selector chain has classes or IDs, those still count.

```svelte
<style>
  /* The element gets the scope class (0-1-0) PLUS the .big class (0-1-0).
     Total specificity 0-2-0 — a plain .big rule in global CSS (0-1-0) loses. */
  p:global(.big) { color: red; }
</style>
```

This is by design: `:global(.big)` opts `.big` out of *scoping*, not out of *specificity*.

## `:global` and the Scope Class on Elements

When a selector contains `:global(...)`, the compiler still walks the surrounding selectors and adds the scope class to them — but **not** to the parts inside `:global(...)`.

```svelte
<style>
  .card :global(.body) p { color: blue; }
</style>
```

Compiled:

```css
.card.svelte-xyz .body p.svelte-xyz { color: blue; }
```

Both `.card` and the trailing `p` get the scope hash. `.body` does not (because it was inside `:global(...)`).

## `:global` and `&` Nesting

You can use CSS nesting with `:global` selectors in the parent position.

```svelte
<style>
  :global(.dark) {
    .card { background: #222; color: #eee; }
  }
</style>
```

Compiled:

```css
.dark .card.svelte-xyz { background: #222; color: #eee; }
```

The `.card` is still scoped; the surrounding `.dark` is not.

## When to Reach for `:global` (Decision Tree)

1. **Can you pass a CSS custom property instead?** If yes, do that. Custom properties are the recommended way to style children.
2. **Are you styling a single element class that's used everywhere?** Use a global CSS file imported once at app entry.
3. **Are you styling something inside `{@html}` content?** Use `:global`.
4. **Are you styling a third-party widget that doesn't accept custom properties?** Use `:global`, scoped to a parent that you control.
5. **Are you writing app-wide resets and tokens?** Use `:global { :root { ... } }` or a global CSS file.
6. **Are you writing a top-level global animation?** Use `@keyframes -global-name`.

## Limitations of `:global`

- **It doesn't compose with the component's other scoped rules.** Once you opt out, you've opted out — the rule will match anywhere the selector matches, not just inside the component.
- **It's easy to leak styles.** A `:global(.btn)` in one component is invisible to other components at edit time but visible at runtime.
- **It defeats encapsulation.** If a child component changes its DOM structure, the rule may silently stop matching.

## `:global` and the `svelte-css-wrapper`

When a component receives `--css-custom-properties` from its parent, the parent renders a `svelte-css-wrapper` element (or `<g>` for SVG) around the component. This wrapper has the inline style carrying the variables.

- The wrapper has `display: contents` by default, so it does not affect layout.
- It *does* affect the `>` (child) combinator — a parent rule like `.parent > *` will match the wrapper, not the actual component.
- `:global` rules can target the wrapper directly if needed: `:global(svelte-css-wrapper)`.

## Summary of Selector Transforms

| Source | Compiled |
|--------|----------|
| `p` | `p.svelte-xyz` |
| `.foo` | `.foo.svelte-xyz` |
| `p .foo` | `p.svelte-xyz .foo.svelte-xyz` (second occurrence wrapped in `:where()`) |
| `:global(body)` | `body` (no scope class) |
| `div :global(strong)` | `div.svelte-xyz strong` |
| `p:global(.big.red)` | `p.svelte-xyz.big.red` |
| `:global { div { ... } }` | `div { ... }` (no scope class) |
| `.a :global { .b { ... } }` | `.a.svelte-xyz .b { ... }` |
| `@keyframes name` | `@keyframes name-svelte-xyz` + all `animation` refs rewritten |
| `@keyframes -global-name` | `@keyframes name` (prefix stripped, refs unchanged) |

## See also

- [examples/global-patterns.md](../examples/global-patterns.md) — practical patterns
- [examples/scoped-keyframes.md](../examples/scoped-keyframes.md) — `-global-` keyframes
- [references/specificity.md](./specificity.md) — how scoped and global rules interact
- [references/scoped-deep.md](./scoped-deep.md) — base scoping mechanism
