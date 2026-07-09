# Global Patterns: `:global(...)` Examples

`:global(...)` opts a selector out of Svelte's scoping mechanism. There are two forms:

- **`:global(selector)`** — applies to a single selector (or a compound selector).
- **`:global { ... }`** — applies to a whole block of selectors.

This file collects the practical patterns: when to reach for global, how to limit its blast radius, and the trade-offs.

## Table of Contents

- [`:global()` on a Single Tag](#global-on-a-single-tag)
- [`:global()` on a Single Class](#global-on-a-single-class)
- [`:global()` Inside a Scoped Selector (Nested Global)](#global-inside-a-scoped-selector-nested-global)
- [`:global()` with a Compound Selector](#global-with-a-compound-selector)
- [`:global { ... }` Block (Multiple Selectors)](#global--block-multiple-selectors)
- [`:global` After a Scoped Prefix](#global-after-a-scoped-prefix)
- [Globals That Reach Up (Into a Slot)](#globals-that-reach-up-into-a-slot)
- [Choosing Between `:global(...)` and CSS Custom Properties](#choosing-between-global-and-css-custom-properties)
- [Common Pitfalls](#common-pitfalls)
- [When NOT to Use `:global`](#when-not-to-use-global)

## `:global()` on a Single Tag

Target an element by tag name globally. The selector receives **no** scope hash.

```svelte
<style>
  :global(body) {
    margin: 0;
    font-family: system-ui, -apple-system, sans-serif;
  }

  :global(html) {
    height: 100%;
  }
</style>
```

Use this for top-level resets that must reach the `<body>` or `<html>` (which live outside any component's scope).

## `:global()` on a Single Class

```svelte
<style>
  :global(.app-shell) {
    display: grid;
    grid-template-columns: 240px 1fr;
    min-height: 100vh;
  }
</style>
```

`.app-shell` is matched anywhere in the document, regardless of scope. Useful for a layout class that several components add to their root.

## `:global()` Inside a Scoped Selector (Nested Global)

The most common form: keep the outer part scoped, opt just the inner part out.

```svelte
<style>
  /* The .card is scoped, but the inner strong is global. */
  .card :global(strong) {
    color: goldenrod;
  }
</style>

<div class="card">
  <p>This <strong>strong</strong> is goldenrod, even when the
     strong comes from a child component or is rendered via {@html}.</p>
</div>
```

In source it reads "any strong inside a card belonging to this component". In compiled CSS it becomes:

```css
.card.svelte-xyz strong { color: goldenrod; }
```

The `strong` is unaffected by Svelte's scoping — important when the `<strong>` is rendered by a third-party library or another component, neither of which has this component's scope hash.

## `:global()` with a Compound Selector

A *compound* selector (`tag.class.class` or `.a.b`) inside `:global(...)` is treated as a single scoped unit on the *element*, but the whole compound bypasses the parent scope walk.

```svelte
<style>
  /* The element MUST be a <p> tag with classes "big" AND "red",
     AND that <p> must be in this component's scope (it gets the scope class). */
  p:global(.big.red) {
    font-weight: bold;
    font-size: 1.25rem;
  }
</style>

<p class="big red">Stands out</p>
```

This is the form to reach for when a library adds a class to an element you own. The element still gets your scope hash, so other scoped rules in this component continue to apply, but the *trigger* class (`.big.red`) doesn't need to exist in your template at compile time.

Compiled:

```css
p.svelte-xyz.big.red { font-weight: bold; font-size: 1.25rem; }
```

## `:global { ... }` Block (Multiple Selectors)

When several rules share the "I want this global" intent, group them.

```svelte
<style>
  :global {
    *, *::before, *::after {
      box-sizing: border-box;
    }

    :root {
      --color-bg: #ffffff;
      --color-text: #111111;
      --color-primary: #ff3e00;
    }

    h1, h2, h3, h4, h5, h6 {
      margin: 0;
      line-height: 1.2;
    }

    a {
      color: var(--color-primary);
    }
  }
</style>
```

Everything inside the block is emitted without a scope hash. Great for top-of-file resets and design tokens.

## `:global` After a Scoped Prefix

The block form also accepts a leading scoped selector — the leading part is scoped, everything after `:global` is not.

```svelte
<style>
  .tooltip :global {
    /* The descendant walk inside is unscoped. */
    .arrow { fill: currentColor; }
    .body  { background: black; color: white; }
  }
</style>
```

Equivalent to:

```svelte
<style>
  .tooltip :global .arrow { fill: currentColor; }
  .tooltip :global .body  { background: black; color: white; }
</style>
```

The nested form is preferred for readability when there are several descendants.

You can also reach up — selectors before `:global` can be scoped ancestors of an unscoped branch.

## Globals That Reach Up (Into a Slot)

Slot content is rendered in the **parent's** scope, not the child's. If you need the child component to style slotted content, use `:global`.

```svelte
<!-- Card.svelte -->
<style>
  :global(.card-slotted) {
    /* Matches .card-slotted anywhere inside the slot. */
    padding: 0.5rem;
  }
</style>

<div class="card">
  <slot />
</div>
```

```svelte
<!-- App.svelte -->
<Card>
  <p class="card-slotted">Styled by Card</p>
</Card>
```

Be careful: `:global(.card-slotted)` will match the class *anywhere in the app*, not just inside the slot. To restrict, scope the surrounding context:

```svelte
<!-- Card.svelte -->
<style>
  .card :global(.card-slotted) {
    padding: 0.5rem;
  }
</style>
```

Now the class only matches when it's a descendant of `.card` (which is scoped to Card).

## Choosing Between `:global(...)` and CSS Custom Properties

The decision tree for styling things that "live below" your component:

| You want to… | Reach for |
|--------------|-----------|
| Pass a color/spacing value down | **CSS custom property** (`--foo`) |
| Restructure layout of a child's children | `:global` (rare, last resort) |
| Style slotted content from the host | `:global` (or `::slotted()`) |
| Apply app-wide resets / tokens | `:global { ... }` in a top-level component |
| Theme a third-party widget | `:global` (or override with custom properties) |

Custom properties are preferred because they don't break encapsulation: the child controls how the value is used. `:global` reaches across boundaries, which couples components together.

## Common Pitfalls

### Forgetting the scope hash on a tag selector you "own"

```svelte
<style>
  /* This p must be in this component's template; otherwise it does nothing. */
  p { color: red; }
</style>
```

If `<p>` lives in a child component, this rule never matches. Either move the `<p>` into this component or use `:global(p)`.

### Using `:global` "just in case"

```svelte
<!-- Anti-pattern -->
<style>
  :global(.btn) { /* ... */ }
</style>
```

This pollutes the global class namespace. If you need to share `.btn` styles across components, use a CSS file imported by both, or use `:global` only inside a wrapper:

```svelte
<style>
  .toolbar :global(.btn) { /* .btn only inside .toolbar */ }
</style>
```

### `:global` order with cascade

A `:global` rule loaded by an "earlier" component can still lose to a later scoped rule, because scoped selectors have higher specificity (0-1-0 extra). If a global reset is being overridden by a scoped component, that's why.

## When NOT to Use `:global`

- **You want a child to render a different color.** Use a custom property. See [css-custom-properties.md](../references/css-custom-properties.md).
- **You're fighting Svelte's scoping.** That's a sign your design wants the styled element to live in this component.
- **You're styling a third-party widget that uses BEM-style classes inside shadow DOM.** Use a custom property exposed by the widget, not `:global` from outside.
- **You want one-off element tweaks for an isolated page.** Just write a global CSS file — no need to put it in a component's `<style>`.

## See also

- [references/global-reference.md](../references/global-reference.md) — full reference of `:global(...)` forms and how they compile
- [references/specificity.md](../references/specificity.md) — when global + scoped rules collide
- [examples/scoped-styles.md](./scoped-styles.md) — basic scoping rules
- [examples/scoped-keyframes.md](./scoped-keyframes.md) — global keyframes via `-global-`
