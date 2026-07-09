# Nested `<style>` Elements Examples

A Svelte component can have **only one top-level `<style>` tag**. But you can put additional `<style>` tags *inside* other elements or logic blocks. These nested style tags are inserted into the DOM as-is — **no scoping, no processing** — and behave like a normal HTML `<style>` element.

This file shows the patterns, trade-offs, and use cases.

## Table of Contents

- [The Rule: One Top-level Style](#the-rule-one-top-level-style)
- [Nested `<style>` — Raw, Unscoped](#nested-style--raw-unscoped)
- [Inside an `{#if}` Block](#inside-an-if-block)
- [Inside an `{#each}` Block (Conditional Per-Item CSS)](#inside-an-each-block-conditional-per-item-css)
- [Inside a Component (Imported Snippet)](#inside-a-component-imported-snippet)
- [`<style>` Inside a Slot](#style-inside-a-slot)
- [When to Use Nested `<style>`](#when-to-use-nested-style)
- [When NOT to Use It](#when-not-to-use-it)
- [`:global` vs Nested `<style>`](#global-vs-nested-style)
- [Server-Rendered HTML and Nested `<style>`](#server-rendered-html-and-nested-style)
- [Common Gotchas](#common-gotchas)

## The Rule: One Top-level Style

```svelte
<script>
  /* one component, one top-level <style> */
</script>

<style>
  /* scoped to this component */
  .foo { color: red; }
</style>
```

A second top-level `<style>` in the same component is a compile error. To add more rules, either add them to the single top-level style or nest them as shown below.

## Nested `<style>` — Raw, Unscoped

```svelte
<div>
  <style>
    /* This CSS is injected into the DOM as-is.
       No scope hash. Selectors reach globally. */
    div { color: red; }
  </style>
  <p>I'm red because the nested <style> targets every <div>.</p>
</div>
```

**Compiled output** (conceptually):

```html
<div>
  <style>div { color: red; }</style>
  <p>...</p>
</div>
```

The `<style>` element ends up in the live DOM, contributing to the document's stylesheets. The browser applies its rules to the entire document — not just the children of the surrounding `<div>`.

## Inside an `{#if}` Block

Styles are only added to the DOM when the block is rendered. Useful for conditionally-inserted style fragments.

```svelte
<script>
  let dark = $state(false);
</script>

<button onclick={() => dark = !dark}>
  Toggle dark
</button>

{#if dark}
  <style>
    :root {
      --bg: #111;
      --text: #f5f5f5;
    }
  </style>
{/if}

<style>
  body {
    background: var(--bg, white);
    color: var(--text, black);
  }
</style>
```

When `dark` is true, the nested `<style>` is added, setting the variables on `:root`. Toggling `dark` to false removes the style and the variables go back to their fallback values.

## Inside an `{#each}` Block (Conditional Per-Item CSS)

Each iteration of `{#each}` produces its own `<style>` tag (if it has one). This is rare but sometimes useful for tiny, scoped style fragments.

```svelte
<script>
  let themes = ['red', 'green', 'blue'];
</script>

{#each themes as color}
  <div class="swatch">
    <style>
      .swatch-{color} { background: {color}; }
    </style>
    <span>{color}</span>
  </div>
{/each}
```

For dynamic theming you almost always want **CSS custom properties** instead — this is purely an illustration of the feature.

## Inside a Component (Imported Snippet)

```svelte
<script>
  import ThemeStyles from './ThemeStyles.svelte';
</script>

<ThemeStyles />

<!-- This works because Svelte allows <style> nested inside component markup. -->
{#if false}
  <style>
    /* never reached, but valid syntax */
    .unused { color: blue; }
  </style>
{/if}
```

## `<style>` Inside a Slot

A child component's slot can include a `<style>` tag. The style is rendered into the DOM as part of the slot content, without scoping.

```svelte
<!-- Modal.svelte -->
<div class="modal">
  <slot />
</div>
```

```svelte
<!-- App.svelte -->
<script>
  let open = $state(true);
</script>

{#if open}
  <Modal>
    <h2>Title</h2>
    <style>
      /* Global, affects everything */
      h2 { color: purple; }
    </style>
  </Modal>
{/if}
```

Use this only when the parent needs to inject one-off global CSS as part of its layout. In most cases, prefer a real global stylesheet.

## When to Use Nested `<style>`

- **Toggling global tokens on/off.** Theme variables that need to disappear entirely when a mode is inactive (e.g. removing a `--dark` set on `:root`).
- **Self-contained third-party integrations.** A component that brings its own CSS and embeds it inline.
- **Legacy migration.** Moving a large CSS block from a static HTML file into a Svelte component without rewriting all selectors as `:global(...)`.
- **Email / SVG with embedded `<style>`.** SVG documents often include `<style>` for fills and animations.
- **One-off dynamic stylesheets** that change at runtime (e.g. user-customizable themes).

## When NOT to Use It

- **You just need a few extra rules.** Put them in the top-level `<style>` block.
- **You want to scope styles to a child component.** Use a regular `<style>` inside that child component, or use `:global` from the parent.
- **You're trying to "import a CSS file."** Use `import './foo.css'` from the `<script>` block — Vite (or your bundler) handles it.
- **You want a single source of truth for design tokens.** Put them in a top-level `:global { :root { ... } }` block, or in a global stylesheet imported once at app entry.

## `:global` vs Nested `<style>`

Both can apply rules globally. Differences:

| Aspect | `:global { ... }` in top-level `<style>` | Nested `<style>` |
|--------|------------------------------------------|------------------|
| Compiled | Stripped of scoping, emitted once | Inserted as-is into DOM |
| Lifecycle | Lives in the document `<head>` (or wherever the compiler puts it) | Lives where it's rendered, removed with the surrounding element |
| Can be toggled | No — present for the component's lifetime | Yes — `{#if}` can add/remove it |
| Performance | One entry in the stylesheet | One `<style>` element per render |
| Specificity battles | Avoid (use a separate stylesheet) | Easy to ignore in cascade calculations |

A simple rule of thumb: **for static global CSS, use `:global` or an imported stylesheet. For dynamic/conditional global CSS, use a nested `<style>` inside an `{#if}`.**

## Server-Rendered HTML and Nested `<style>`

Nested styles are rendered into the SSR output. They end up in the HTML response just like a hand-written `<style>` would, and the browser processes them on hydration.

```svelte
<!-- Component.svelte -->
{#if dark}
  <style>:root { --bg: #111; }</style>
{/if}
```

```html
<!-- Server-rendered HTML when dark = true -->
<style>:root { --bg: #111; }</style>
```

After hydration, Svelte's runtime takes over the `{#if}` and the `<style>` is added/removed as the variable changes.

## Common Gotchas

### The nested style isn't scoped

It's global. The selector `div { color: red }` in a nested style affects every `<div>` on the page, not just the one wrapping the style tag.

### Hot-reload and dev iteration

A nested `<style>` in a frequently re-rendered block can cause a flood of style elements during dev. Prefer a top-level style during development; only nest when you actually need the dynamic lifecycle.

### Style tag order

Multiple nested styles apply in DOM order — *last one wins* for equal-specificity rules. Be deliberate about where the style lands.

### Duplicate `@keyframes` declarations

If a nested `<style>` declares `@keyframes foo` and another component (or another nested style) also declares it, the last one wins. Consider `-global-` keyframes in the top-level style to avoid this.

## See also

- [references/scoped-deep.md](../references/scoped-deep.md) — when styles don't apply, scoping internals
- [examples/global-patterns.md](./global-patterns.md) — `:global(...)` patterns
- [examples/scoped-styles.md](./scoped-styles.md) — base scoping rules
