# Scoped Styles Deep Dive

A comprehensive reference on Svelte's styling scope mechanism.

## How Scoping Hash Is Generated

Svelte's compiler generates a unique hash for each component to scope styles. The hash is derived from:

1. **Component's unique ID** - A hash of the component's file path and content
2. **Style block contents** - Ensures different styles get different hashes

### Example Transformation

**Source component** (`Card.svelte`):
```svelte
<style>
  .title { color: blue; }
  .content { font-size: 1rem; }
</style>

<h1 class="title">Hello</h1>
<p class="content">World</p>
```

**Compiled output**:
```html
<h1 class="title svelte-xyz123">Hello</h1>
<p class="content svelte-xyz123">World</p>
```

### Hash Format

- Format: `svelte-[a-z0-9]{5,7}` (e.g., `svelte-abc12`, `svelte-xyz98765`)
- Component-level unique: Two components in the same app will have different hashes
- Deterministic: Same component with same styles produces same hash

### Specificity Impact

Scoped selectors have **0-1-0 specificity** (one additional class):

```css
/* This */
p { color: red; }

/* Becomes this (0-1-0 specificity) */
p.svelte-xyz123 { color: red; }
```

This means scoped `p` selector (0-1-0) beats global `p` selector (0-0-1).

---

## Pseudo-classes Behavior

Pseudo-classes work seamlessly with scoped styles. The scoping hash is added to the element, and pseudo-classes apply on top.

### Common Pseudo-classes

```svelte
<style>
  .btn:hover {
    background: #ff3e00;
  }

  .item:first-child {
    border-top: none;
  }

  .item:last-child {
    border-bottom: none;
  }

  .row:nth-child(even) {
    background: #f5f5f5;
  }

  .input:focus {
    outline: 2px solid blue;
  }

  .box:active {
    transform: scale(0.95);
  }

  .link:visited {
    color: purple;
  }
</style>
```

### Pseudo-classes with Combinators

```svelte
<style>
  /* Scoped hover on child elements */
  .card:hover .card-title {
    color: #ff3e00;
  }

  /* Scoped first-letter */
  .intro::first-letter {
    font-size: 2rem;
    font-weight: bold;
  }

  /* Scoped selection */
  .article::selection {
    background: #ff3e00;
    color: white;
  }
</style>
```

---

## :global() Syntax Variants

### Single Selector

```svelte
<style>
  /* Global element */
  :global(body) { margin: 0; }

  /* Global class */
  :global(.my-class) { color: red; }

  /* Global attribute selector */
  :global([data-theme="dark"]) { background: black; }
</style>
```

### Nested Global

Target elements inside scoped context but without scoping hash:

```svelte
<style>
  /* Targets strong inside any div in this component */
  div :global(strong) { color: goldenrod; }
</style>

<div>
  <!-- This strong gets styled -->
  <strong>Golden</strong>
</div>
```

### Conditional Global

```svelte
<style>
  /* Global class with condition */
  p:global(.important) { font-weight: bold; }
</style>

<p class="important">Bold paragraph</p>
```

### :global {...} Block

Multiple selectors without repeating `:global`:

```svelte
<style>
  :global {
    /* Multiple global selectors */
    body, html {
      height: 100%;
      margin: 0;
    }

    /* Global with nested selectors */
    .theme-dark {
      --bg: #111;
      --text: #eee;
    }

    /* Deep nested global */
    div :global {
      .foo .bar { color: red; }
    }
  }
</style>
```

---

## Scoped Styles Not Affecting Children

By design, scoped styles **do not pierce component boundaries**.

### Parent Component

```svelte
<!-- Parent.svelte -->
<style>
  .container {
    padding: 1rem;
  }

  /* This WILL affect Child.svelte's root element */
  .container > :global(*) {
    border: 1px solid red;
  }
</style>

<div class="container">
  <Child />
</div>
```

### Child Component

```svelte
<!-- Child.svelte -->
<style>
  .root {
    /* This only affects Child's internal elements */
    padding: 0.5rem;
  }
</style>

<div class="root">
  <!-- Styles from Parent.svelte cannot reach inside here -->
  <p>Internal paragraph</p>
</div>
```

**Result**:
- Parent's `.container` styles apply to `<Child>` wrapper
- Parent cannot style `.root` or `p` inside `Child`
- Child's `.root` styles apply to its internal div

---

## How to Affect Child Components

Two primary strategies exist for styling child components from a parent.

### Strategy 1: CSS Custom Properties (Recommended)

Pass styling via CSS custom properties that child components read.

**Parent.svelte**:
```svelte
<style>
  .parent-container {
    /* These variables are inherited by children */
    --button-bg: #ff3e00;
    --button-color: white;
    --card-padding: 1rem;
  }
</style>

<div class="parent-container">
  <Button>Click me</Button>
  <Card />
</div>
```

**Button.svelte**:
```svelte
<style>
  .btn {
    /* Read the custom property passed by parent */
    background: var(--button-bg, #ccc);
    color: var(--button-color, #333);
    padding: var(--button-padding, 0.5rem 1rem);
  }
</style>

<button class="btn">
  <slot />
</button>
```

**Advantages**:
- Child controls how it uses the values
- Cascades through component tree
- Dynamic updates via `$state`
- No global pollution

### Strategy 2: :global() (Use Sparingly)

Use `:global()` to bypass scoping entirely.

**Parent.svelte**:
```svelte
<style>
  /* This affects ALL .child-text elements in the app */
  .wrapper :global(.child-text) {
    color: purple;
  }
</style>

<div class="wrapper">
  <Child />
</div>
```

**Child.svelte**:
```svelte
<div class="child-text">
  <!-- Will be purple because Parent used :global -->
  I am styled from parent
</div>
```

**Disadvantages**:
- Affects global scope
- Potential conflicts
- Harder to maintain
- Defeats style isolation

### Comparison Table

| Aspect | CSS Custom Properties | :global() |
|--------|----------------------|-----------|
| Scope | Inherited only | Global |
| Maintenance | Child controls usage | Parent controls targeting |
| Conflicts | Unlikely | Possible |
| Dynamic | Yes ($state) | Limited |
| Type Safety | Full | None |
| Recommended | Yes | Sparingly |

---

## Keyframe Scoping

Keyframe names are automatically scoped, but you can make them global.

### Scoped Keyframes

```svelte
<style>
  @keyframes bounce {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-20px); }
  }

  .animated {
    animation: bounce 1s;
    /* Actually runs: bounce svelte-xyz123 1s */
  }
</style>
```

### Global Keyframes (-global- prefix)

```svelte
<style>
  /* The -global- prefix makes keyframe name global */
  @keyframes -global-fade-in {
    from { opacity: 0; }
    to { opacity: 1; }
  }

  @keyframes -global-slide-up {
    from { transform: translateY(20px); opacity: 0; }
    to { transform: translateY(0); opacity: 1; }
  }
</style>

<div style="animation: fade-in 0.5s">
  <!-- Can be referenced by other components as just "fade-in" -->
</div>
```

**Note**: The `-global-` prefix is stripped during compilation.

---

## {@html} and Scoped Styles

Content rendered via `{@html}` is **not scoped** because it has no scope hash.

```svelte
<script>
  let htmlContent = '<p class="highlight">Dynamic content</p>';
</script>

<style>
  /* This will NOT affect the .highlight paragraph */
  .highlight { color: red; }
</style>

{@render htmlContent}
```

**Solution**: Use `:global()` if you control the HTML content:

```svelte
<style>
  :global(.highlight) { color: red; }
</style>
```

**Warning**: Be cautious with user-generated content to avoid XSS vulnerabilities.

---

## Multiple Style Blocks

Svelte only allows **one top-level `<style>` block**, but you can nest `<style>` elements inside HTML structures.

### Nested Style (No Scoping)

```svelte
<div>
  <!-- This style tag is inserted directly into DOM, no scoping -->
  <style>
    .raw { color: green; }
  </style>
  <p class="raw">I am green without hash</p>
</div>
```

**Use cases**:
- Third-party integrations
- Email templates
- Legacy code migration
