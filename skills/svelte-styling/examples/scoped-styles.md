# Scoped Styles Examples

This file covers all scoped styling patterns in Svelte 5.

## Table of Contents

- [Basic Scoped Style](#basic-scoped-style)
- [Pseudo-classes](#pseudo-classes-hover-first-child-etc)
- [:global() Single Selector](#global-single-selector)
- [:global() Nested](#global-nested-parent-wraps-child)
- [:global() Block Syntax](#global-block-syntax)
- [CSS Custom Properties Passing](#css-custom-properties--var-passing-from-parent)
- [CSS Custom Properties Reading](#css-custom-properties-reading-inside-component)
- [class Attribute Object Form](#class-attribute-with-object-form)
- [class Attribute Array Form](#class-attribute-with-array-form)
- [Tailwind Integration](#tailwind-integration)
- [Combining Multiple Patterns](#combining-multiple-patterns)
- [Specificity: Scoped vs Global](#specificity-scoped-vs-global)
- [Specificity: :where() Trick](#specificity-where-trick-for-lower-scope-specificity)
- [When Styles Don't Apply](#when-styles-dont-apply)
- [Compiled CSS: What the Compiler Emits](#compiled-css-what-the-compiler-emits)
- [Class-only / Tag-only Selectors in Scope](#class-only-and-tag-only-scoped-selectors)
- [Slot Children and Style Pierce](#slot-children-and-style-pierce)
- [Svelte Wrapper Element Caveat](#svelte-css-wrapper-element)

## Basic Scoped Style

Styles in `<style>` blocks are automatically scoped to the component.

```svelte
<script>
  let count = $state(0);
</script>

<style>
  .card {
    padding: 1rem;
    border: 1px solid #ddd;
    border-radius: 8px;
  }

  .title {
    font-size: 1.25rem;
    font-weight: 600;
  }
</style>

<div class="card">
  <h1 class="title">Count: {count}</h1>
  <button onclick={() => count++}>Increment</button>
</div>
```

**Compiled output** (simplified):
```html
<div class="card svelte-123abc">
  <h1 class="title svelte-456def">Count: 0</h1>
  <button>Increment</button>
</div>
```

---

## Pseudo-classes (:hover, :first-child, etc.)

Pseudo-classes work naturally with scoped styles.

```svelte
<style>
  .btn {
    padding: 0.5rem 1rem;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    transition: background 0.2s;
  }

  .btn:hover {
    background: #ff3e00;
    color: white;
  }

  .btn:active {
    transform: scale(0.98);
  }

  .list {
    list-style: none;
    padding: 0;
  }

  .list-item:first-child {
    border-top: none;
  }

  .list-item:last-child {
    border-bottom: none;
  }

  .list-item:nth-child(odd) {
    background: #f5f5f5;
  }

  .input:focus {
    outline: 2px solid #ff3e00;
    outline-offset: 2px;
  }
</style>

<button class="btn">Hover me</button>

<ul class="list">
  <li class="list-item">First item</li>
  <li class="list-item">Second item</li>
  <li class="list-item">Third item</li>
</ul>

<input class="input" placeholder="Focus me" />
```

---

## :global() Single Selector

Use `:global()` to apply styles outside the component's scope.

```svelte
<style>
  /* Affects the document body */
  :global(body) {
    margin: 0;
    font-family: system-ui, sans-serif;
  }

  /* Global keyframes - note the -global- prefix */
  @keyframes -global-fade-in {
    from { opacity: 0; }
    to { opacity: 1; }
  }

  .modal {
    animation: fade-in 0.3s ease-out;
  }

  /* Nested global - affects <strong> inside any div */
  div :global(strong) {
    color: goldenrod;
  }

  /* Global with conditional class */
  p:global(.important) {
    font-weight: bold;
  }
</style>

<div class="modal">
  <p class="important">This paragraph gets the global .important class</p>
  <div><strong>This strong text is goldenrod</strong></div>
</div>
```

---

## :global() Nested (Parent Wraps Child)

Parent components can use `:global()` to style children in global context.

```svelte
<!-- Parent.svelte -->
<style>
  /* This targets .child-text anywhere inside .parent-container */
  .parent-container :global(.child-text) {
    color: purple;
    font-style: italic;
  }
</style>

<div class="parent-container">
  <!-- The child's text will be purple -->
  <ChildComponent />
</div>
```

```svelte
<!-- ChildComponent.svelte -->
<div class="child-text">
  I will be purple because Parent uses :global() to reach me
</div>
```

**Note**: This approach affects ALL `.child-text` elements inside `.parent-container`, even from other components.

---

## :global() Block Syntax

Use block syntax for multiple global selectors.

```svelte
<style>
  :global {
    /* Reset styles */
    *, *::before, *::after {
      box-sizing: border-box;
    }

    /* Theme variables */
    :root {
      --primary: #ff3e00;
      --secondary: #40e0d0;
    }

    /* Multiple selectors */
    h1, h2, h3 {
      margin: 0;
    }

    /* Nested global block */
    .theme-dark {
      --bg: #111;
      --text: #eee;
    }

    /* Deep nested global */
    div :global {
      .foo .bar {
        color: red;
      }
    }
  }
</style>

<div class="theme-dark">
  <h1>Global styles applied</h1>
</div>
```

---

## CSS Custom Properties (--var) Passing from Parent

Pass CSS custom properties to child components via attributes.

```svelte
<!-- Parent.svelte -->
<style>
  .slider-container {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }
</style>

<div class="slider-container">
  <Slider
    bind:value={position}
    --track-color="#ff3e00"
    --thumb-color="rgb(255 0 0)"
    --track-width="4px"
  />
</div>
```

```svelte
<!-- Slider.svelte -->
<script>
  let { value = $bindable(0), ...props } = $props();
</script>

<style>
  .track {
    background: var(--track-color, #aaa);
    height: var(--track-width, 2px);
    border-radius: 2px;
  }

  .thumb {
    background: var(--thumb-color, blue);
    width: 16px;
    height: 16px;
    border-radius: 50%;
    transform: translateY(calc(var(--track-width, 2px) / -2));
  }
</style>

<div class="track">
  <div class="thumb" style="top: {value}px"></div>
</div>
```

---

## CSS Custom Properties Reading Inside Component

Read CSS custom properties directly in your component's styles.

```svelte
<script>
  let { variant = 'primary', ...props } = $props();
</script>

<style>
  .button {
    /* Read custom properties with fallback defaults */
    background: var(--button-bg, #ff3e00);
    color: var(--button-color, white);
    padding: var(--button-padding, 0.5rem 1rem);
    border: var(--button-border, none);
    border-radius: var(--button-radius, 4px);
    font-size: var(--button-font-size, 1rem);
  }

  /* Variant overrides via CSS custom properties */
  :global(.dark) .button {
    --button-bg: #333;
    --button-color: #fff;
  }
</style>

<button class="button" {...props}>
  <slot />
</button>
```

**Usage**:
```svelte
<Button --button-bg="green">Green Button</Button>
<Button --button-radius="9999px">Pill Button</Button>
```

---

## class Attribute with Object Form

Use object syntax for conditional classes (clsx-style).

```svelte
<script>
  let { isActive = false, isLarge = false } = $props();
</script>

<style>
  .badge {
    display: inline-flex;
    align-items: center;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    font-size: 0.875rem;
  }

  .active {
    background: #22c55e;
    color: white;
  }

  .large {
    padding: 0.5rem 1rem;
    font-size: 1rem;
  }
</style>

<!-- isActive=true adds 'active', isLarge=true adds 'large' -->
<div class={{ active: isActive, large: isLarge }}>
  Badge Content
</div>
```

---

## class Attribute with Array Form

Use array syntax for multiple conditional classes.

```svelte
<script>
  let isFaded = $state(true);
  let isScaled = $state(false);
</script>

<style>
  .card {
    transition: all 0.3s ease;
  }

  .saturate-0 {
    filter: saturate(0);
  }

  .opacity-50 {
    opacity: 0.5;
  }

  .scale-200 {
    transform: scale(2);
  }
</style>

<!-- Combines conditional classes -->
<div class={[
  isFaded && 'saturate-0 opacity-50',
  isScaled && 'scale-200',
  'card'
]}>
  Array Class Card
</div>
```

---

## Tailwind Integration

See [tailwind.md](./tailwind.md) for comprehensive Tailwind examples.

### Basic Tailwind Setup

```svelte
<script>
  let { class: cls = '' } = $props();
</script>

<style>
  .btn {
    /* Tailwind utility classes applied via class prop */
  }
</style>

<button class={['px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600', cls]}>
  <slot />
</button>
```

### Passing Tailwind Classes to Child Components

```svelte
<!-- Parent.svelte -->
<script>
  let { class: cls = '' } = $props();
</script>

<Card class={['shadow-lg rounded-lg p-4', cls]}>
  <p>Card with Tailwind styling</p>
</Card>
```

```svelte
<!-- Card.svelte -->
<script>
  let { class: cls = '', children } = $props();
</script>

<div class={['bg-white border border-gray-200', cls]}>
  {@render children()}
</div>
```

---

## Combining Multiple Patterns

```svelte
<script>
  let { variant = 'default' } = $props();

  let isHovered = $state(false);
  let isLoading = $state(false);
</script>

<style>
  .card {
    padding: 1rem;
    border-radius: 8px;
    transition: all 0.2s;
  }

  /* Variant classes */
  .card.primary {
    background: var(--primary-color, #ff3e00);
    color: white;
  }

  .card.secondary {
    background: #f5f5f5;
    color: #333;
  }

  /* Hover state */
  .card:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  }

  /* Loading state */
  .card.loading {
    opacity: 0.7;
    pointer-events: none;
  }
</style>

<div
  class={[
    'card',
    variant,
    isHovered && 'hover',
    isLoading && 'loading',
    $props().class
  ]}
  onmouseenter={() => isHovered = true}
  onmouseleave={() => isHovered = false}
>
  <slot />
</div>
```

---

## Specificity: Scoped vs Global

Every scoped selector picks up an **extra class** (the scope hash), so its specificity is **0-1-0** higher than the same selector in a global stylesheet. This means a scoped `p` rule beats a global `p` rule — even when the global stylesheet is loaded *after* the component.

```svelte
<!-- Card.svelte -->
<style>
  /* 0-1-1 specificity (one tag + one class).
     This wins over the global p {} below. */
  p { color: burlywood; }
</style>

<p>This paragraph is burlywood, not red.</p>
```

```css
/* global.css (loaded later) */
p { color: red; }
```

Order of the stylesheet does not matter — the scoped class tips the cascade.

> **Tip:** If you want a *global* stylesheet to be able to override a scoped rule, use `:where(.svelte-xyz123)` (see next section) or be more specific in the global rule.

---

## Specificity: `:where()` Trick for Lower Scope Specificity

The compiler is smart: when a scope class must be added to a selector *multiple times*, only the **first** occurrence actually contributes to specificity. All subsequent occurrences are wrapped in `:where(.svelte-xyz123)`, which has **zero** specificity.

```svelte
<style>
  /* Written:    .card .title
     Compiled:   .card.svelte-xyz123 :where(.svelte-xyz123) .title.svelte-xyz123
     The middle :where() doesn't add specificity. */
  .card .title { font-weight: 700; }
</style>
```

The practical consequence is that **duplicate scope classes don't compound**. Whether the selector has one or many class occurrences, it only counts as +0-1-0 once.

This is what lets you write deeply nested selectors (`.a .b .c`) without runaway specificity — a global `.title { font-weight: 400 }` in a low-specificity reset can still win.

---

## When Styles Don't Apply

A common source of confusion: a scoped selector "looks right" but does nothing. Check these cases:

### 1. The element is rendered by a child component

A parent cannot reach into a child component with its scoped selector — the child has its own scope hash.

```svelte
<!-- Parent.svelte -->
<style>
  /* Does NOT style the <p> inside Child */
  .container p { color: red; }
</style>

<div class="container">
  <Child />
</div>
```

```svelte
<!-- Child.svelte -->
<div class="container">  <!-- different scope -->
  <p>This stays default color</p>
</div>
```

**Fix:** Use CSS custom properties (preferred) or `:global()`.

### 2. The class is added programmatically

If a library adds `class="foo"` to your element at runtime, the scoped `.foo` rule still works (the compiler adds the scope hash to the *element*, not the class). But if the library only adds a class to a child it renders internally, scoped styles won't reach it.

### 3. `{@html ...}` content

Content rendered via `{@html}` has no scope hash. Scoped selectors won't apply — use `:global(...)` instead (and never do this with untrusted input).

### 4. The element doesn't actually exist in the template

The compiler only adds the scope hash to elements that match a selector **and** that exist in the template. Dead styles get tree-shaken.

### 5. The selector is more specific elsewhere

If a parent already has a more specific selector that matches (e.g. `body .container p`), the scoped `.container p` (0-2-1) loses to `body .container p` (0-2-1)... actually they tie, and source order wins. Use `:global()` or add specificity deliberately.

---

## Compiled CSS: What the Compiler Emits

A useful mental model: think of scoping as a **two-step transform**.

1. **Selector transform** — every selector that targets a real element gets the scope class appended.
2. **Element transform** — every element in the template gets the scope class added to its `class` attribute.

```svelte
<!-- Source -->
<style>
  .title { color: blue; }
  p { font-size: 1rem; }
</style>

<h1 class="title">Hi</h1>
<p>Hello</p>
```

```js
// Compiled CSS (inside <head> or extracted)
.svelte-abc123 .title, .title.svelte-abc123 { color: blue; }
p.svelte-abc123 { font-size: 1rem; }
```

```html
<!-- Compiled DOM -->
<h1 class="title svelte-abc123">Hi</h1>
<p class="svelte-abc123">Hello</p>
```

Notice: the `.title` rule only matches if **either** the ancestor or the element itself has the scope class — that's a quirk of how Svelte walks parent selectors. In practice, the element class is always added, so it matches.

---

## Class-only and Tag-only Scoped Selectors

You can write both **class** and **tag** selectors in scoped CSS. Both receive the scope hash.

```svelte
<style>
  /* Both get scope class added */
  p { line-height: 1.5; }
  .lead { font-size: 1.25rem; }
  a { color: tomato; }
  ul, ol { padding-left: 1.5rem; }
</style>

<p class="lead">A lead paragraph</p>
<a href="...">Link</a>
<ul><li>One</li></ul>
```

**Gotcha:** if your component has zero `<a>` elements in its template, the `a` rule still compiles, but the compiler may warn or strip it. The compiler only adds the scope class to elements that **actually appear** in the template.

---

## Slot Children and Style Pierce

Slotted children belong to the *parent's* scope, not the child's. This is intentional: the parent controls how slotted content is styled.

```svelte
<!-- Card.svelte -->
<style>
  /* This DOES style the slot child, because slot content
     is rendered in the parent's scope. */
  .card ::slotted(p) {
    color: blue;
  }
</style>

<div class="card">
  <slot />
</div>
```

```svelte
<!-- App.svelte -->
<Card>
  <p>Slotted paragraph — styled by Card's CSS</p>
</Card>
```

For more targeted slot styling, use `::slotted(selector)` from the parent. Note that `::slotted()` only accepts a compound selector and the *outermost* element.

---

## Svelte CSS Wrapper Element

When you pass `--css-custom-properties` to a component, Svelte wraps the component in a `svelte-css-wrapper` element (or `<g>` for SVG). This wrapper has the inline `style` attribute carrying the custom properties.

```svelte
<!-- Source -->
<Slider --track-color="black" />
```

```svelte
<!-- Compiled -->
<svelte-css-wrapper style="display: contents; --track-color: black;">
  <Slider />
</svelte-css-wrapper>
```

Why this matters:

- The wrapper is a real DOM element. CSS selectors like `> .slider` (child combinator) on the parent **break** because the slider is no longer a direct child.
- For most cases `display: contents` makes the wrapper invisible to layout — but `getBoundingClientRect` and `>` combinators still see it.
- The `svelte-css-wrapper` element itself is not styled; it exists only to host the CSS variables.

```svelte
<!-- This selector will NOT match Slider directly -->
<div class="layout">
  <Slider />
</div>

<style>
  .layout > * { padding: 1rem; }  /* matches svelte-css-wrapper, not Slider */
</style>
```

Workarounds: target a class on the wrapper's parent, or use a descendant combinator (`.layout .slider`).

---

## See also

- [scoped-keyframes.md](./scoped-keyframes.md) — `@keyframes` naming and the `-global-` prefix
- [global-patterns.md](./global-patterns.md) — `:global(...)` single, nested, block
- [nested-style.md](./nested-style.md) — raw `<style>` tags inside the template
- [style-directive.md](./style-directive.md) — `style:color`, `style:--var`, `|important`

