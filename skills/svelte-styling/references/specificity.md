# Specificity Reference

A complete reference on how scoped styles interact with the CSS cascade, `:where()`, `:is()`, `:not()`, and override strategies.

## The Specificity Bump from Scoping

Every scoped selector receives an extra class — the component's scope hash. As a result, its **specificity is 0-1-0 higher** than the same selector in a global stylesheet.

```svelte
<!-- Card.svelte -->
<style>
  /* 0-0-1 (just the tag) in the source, but becomes 0-1-1 with the scope class. */
  p { color: burlywood; }
</style>
```

```css
/* global.css */
/* 0-0-1 — loses to the scoped rule. */
p { color: red; }
```

Even when the global stylesheet is loaded *after* the component, the scoped rule wins. The cascade order doesn't help a global rule that has lower specificity.

## How the Scope Class Is Added

The compiler walks each selector and adds the scope class to the **last** compound selector (the part that matches the actual element). The earlier parts of the selector are *also* rewritten to include the scope class, but with a twist.

```svelte
<style>
  /* Source: */
  .card .title { color: red; }
</style>
```

Compiles to:

```css
.card.svelte-xyz123 :where(.svelte-xyz123) .title.svelte-xyz123 {
  color: red;
}
```

- **First occurrence** of the scope class: actual `.svelte-xyz123` — contributes to specificity.
- **Subsequent occurrences** (in the ancestor chain): wrapped in `:where(.svelte-xyz123)` — zero specificity.

The net effect: regardless of how deep the selector is, the scope class only adds **+0-1-0** to the specificity, never more. A `.card .title .body .thing` selector still only beats global `.thing` (0-1-0) and not by runaway amounts.

## Specificity Reference

| Selector | Specificity (a-b-c) | Notes |
|----------|---------------------|-------|
| `p` (global) | 0-0-1 | One element |
| `p.svelte-xyz` (scoped) | 0-1-1 | One class + one element |
| `.foo .bar` (global) | 0-2-0 | Two classes |
| `.foo.svelte-xyz .bar.svelte-xyz` (scoped) | 0-3-0 (with `:where()` for non-last) | Three classes, but the middle one is in `:where()` → 0-2-0 effective |
| `#app p` | 1-0-1 | One ID + one element |
| `p:global(.big)` | 0-2-0 | Element + scope class + `.big` |
| `:global(body)` | 0-0-1 | No scope class, no class selectors |

## Specificity and `:global(...)`

`:global(...)` does **not** lower specificity. It only opts a selector out of *scoping*. The class inside `:global(...)` still counts.

```svelte
<style>
  /* The :global(.big) class is part of the selector's specificity.
     The element still gets the scope class. */
  p:global(.big) { color: red; }
</style>
```

Specificity of the compiled rule: 0-2-0 (`.svelte-xyz` + `.big`). To match a global `.big { color: blue; }` (0-1-0), the scoped rule wins.

If you want `:global(...)` to have lower specificity, wrap the whole rule in `:where()`:

```svelte
<style>
  :where(p:global(.big)) { color: red; }
</style>
```

`:where()` makes the entire selector list zero-specificity, which is rarely what you want — but it's the lever for "I want this rule to be the lowest priority."

## Specificity and `:where()`

`:where(...)` accepts a selector list and contributes **zero** specificity. Useful for writing low-priority defaults.

```svelte
<style>
  /* Zero specificity — every other rule wins. */
  :where(.card) { padding: 0; }

  /* Higher specificity — overrides the :where() above. */
  .card.featured { padding: 2rem; }
</style>
```

The Svelte compiler uses `:where()` internally to wrap extra scope class occurrences, so deeply nested scoped selectors don't accumulate specificity.

## Specificity and `:is()` / `:not()`

`:is(...)` and `:not(...)` take the **specificity of their most specific argument**. They do not "reset" specificity.

```svelte
<style>
  /* Specificity of this rule = specificity of the most specific selector
     inside :is(), which is #featured = 1-0-0. Plus the scope class. */
  :is(p, #featured) { color: red; }
</style>
```

In a scoped rule, the compiler still adds the scope class once to the outermost compound.

## Override Strategies

When a global stylesheet rule should override a scoped rule, the options are:

### 1. Add specificity in the global rule

```css
/* global.css — match or exceed the scoped specificity */
body p { color: red; }   /* 0-0-2 vs scoped 0-1-1 → still loses */
html body p { color: red; }  /* 0-0-3 vs 0-1-1 → wins */
```

### 2. Use `:where()` in the scoped rule

```svelte
<!-- Card.svelte -->
<style>
  /* Wrap the scope class in :where() to keep specificity at 0-0-1 */
  :where(.svelte-xyz) p { color: red; }
</style>
```

You can't do this directly — the scope class is added by the compiler. But you can put `:where()` around your own selectors:

```svelte
<style>
  :where(p) { color: red; }
</style>
```

A scoped rule on a single `p` is 0-1-1. Wrapped in `:where()`, the element's part is zeroed — leaving just the scope class: 0-1-0. Still higher than a global `p` (0-0-1), but lower than a global `.foo p` (0-1-1), so the global rule can win on source order.

### 3. Move the rule to a `:global` block

```svelte
<style>
  :global {
    .card p { color: red; }   /* emitted at 0-1-1 — same as a normal global rule */
  }
</style>
```

A `:global` block emits selectors without the scope class, restoring baseline specificity.

### 4. Use a global stylesheet

```css
/* global.css */
.card p { color: red !important; }
```

`!important` flips the cascade order entirely (within the same origin). But use this as a last resort — `!important` is hard to maintain.

### 5. Use `:where()` in the *global* rule

```css
/* global.css — keep your overrides low-specificity on purpose */
:where(.card) p { color: red; }
```

If you also need the rule to be overridable by other styles, `:where()` makes it sit at the bottom of the cascade.

## Specificity and Inline Styles

Inline styles (`style="..."` attribute) win over any rule in a stylesheet — *except* `!important` rules in the stylesheet.

The `style:` directive sets inline styles. So `style:color="red"` wins over a scoped `.foo { color: blue; }` rule. It also wins over a `style="color: blue"` attribute on the same element (the directive takes precedence, even if the `style` attribute has `!important`).

```svelte
<!-- Directive wins over the style attribute, even with !important -->
<div style:color="red" style="color: blue !important">Red</div>
```

This is unique to `style:` — it's the only place inline styles out-cascade inline `!important`.

## Specificity and the `svelte-css-wrapper`

When a parent passes `--css-custom-properties` to a child, the child is wrapped in a `svelte-css-wrapper` (or `<g>` for SVG) carrying the inline style. The wrapper's own specificity in the cascade is 0 (it's an element with no class or ID), but its `display: contents` makes it invisible to layout.

Specificity gotchas with the wrapper:

- The wrapper has no class. If your scoped CSS targets `.child`, the rule needs the wrapper's *parent* to have the right structure — `.child` won't be matched via the wrapper if the wrapper is between them.
- Use descendant combinators (`.parent .child`) not child combinators (`.parent > .child`) for components that accept custom properties.

## Summary of Override Order

From lowest to highest priority:

1. Browser defaults.
2. Global stylesheet rules (low specificity, normal source order).
3. Global stylesheet rules (higher specificity).
4. Scoped rules (with the +0-1-0 bump from the scope class).
5. `:global(...)` rules inside a component's `<style>` (baseline specificity, no scope class).
6. Inline `style="..."` attributes.
7. `style:` directives (set inline styles, win over `style=""` even with `!important`).
8. `!important` rules in stylesheets (flip the cascade order).
9. `!important` inline styles.
10. `style:|important` directives (the highest priority short of `!important` in user-agent stylesheets).

## See also

- [examples/scoped-styles.md](../examples/scoped-styles.md) — base scoping and specificity examples
- [examples/global-patterns.md](../examples/global-patterns.md) — `:global(...)` and override patterns
- [references/global-reference.md](./global-reference.md) — full `:global(...)` reference
- [references/scoped-deep.md](./scoped-deep.md) — base scoping mechanism
