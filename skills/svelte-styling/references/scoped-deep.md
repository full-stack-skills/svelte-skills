# Scoped Styles Deep Dive

A comprehensive reference on Svelte's styling scope mechanism.

## Table of Contents

- [How Scoping Hash Is Generated](#how-scoping-hash-is-generated)
- [Specificity Impact](#specificity-impact)
- [The `:where()` Trick](#the-where-trick)
- [Pseudo-classes Behavior](#pseudo-classes-behavior)
- [:global() Syntax Variants](#global-syntax-variants)
- [Scoped Styles Not Affecting Children](#scoped-styles-not-affecting-children)
- [How to Affect Child Components](#how-to-affect-child-components)
- [Keyframe Scoping](#keyframe-scoping)
- [Scoped Keyframes: Full Reference](#scoped-keyframes-full-reference)
- [Global Keyframes (`-global-` prefix)](#global-keyframes--global--prefix)
- [{@html} and Scoped Styles](#html-and-scoped-styles)
- [Multiple Style Blocks](#multiple-style-blocks)
- [Nested `<style>` Elements (No Scoping)](#nested-style-elements-no-scoping)
- [The Scope Class in the Cascade](#the-scope-class-in-the-cascade)
- [Edge Cases and Gotchas](#edge-cases-and-gotchas)

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

---

## Scoped Keyframes: Full Reference

`@keyframes` declared in a scoped `<style>` block is renamed the same way classes are — the compiler suffixes the name with a hash. Every `animation` reference in the same block is rewritten to match.

### How the Rewriter Walks Selectors

For a scoped rule like:

```svelte
<style>
  @keyframes wiggle { 0% { transform: rotate(0); } 100% { transform: rotate(5deg); } }
  .thing { animation: wiggle 0.3s; }
</style>
```

The compiled CSS is effectively:

```css
@keyframes wiggle-svelte-abc123 { 0% { transform: rotate(0); } 100% { transform: rotate(5deg); } }
.thing.svelte-abc123 { animation: wiggle-svelte-abc123 0.3s; }
```

The exact suffix form (`svelte-abc123` here) varies across Svelte versions, but the contract is: **the same hash is used for both the class and the keyframe name within a component**, so `animation: wiggle` always finds the right `@keyframes` rule.

### Multi-step Animation Shorthand

`animation` is a shorthand. The rewriter only changes the value of the `animation-name` part — other longhand properties (`animation-duration`, `animation-timing-function`, etc.) are untouched.

```svelte
<style>
  @keyframes wiggle { /* ... */ }
  .thing { animation: wiggle 0.3s ease-in-out 0.1s infinite alternate; }
</style>
```

The compiled rule has `animation: wiggle-svelte-xyz 0.3s ease-in-out 0.1s infinite alternate;` — only the first token (the name) is rewritten.

### Where the Rewriter Does *Not* Apply

The rewriter only touches the same component's scoped CSS. It does **not** rewrite:

- `style="animation: wiggle ..."` attributes
- Inline JS: `element.style.animation = 'wiggle 0.3s'`
- Selectors in `:global { ... }` blocks
- Selectors in nested `<style>` tags (which are inserted as-is)

In all those cases, use `@keyframes -global-name` so the unscoped name is what the browser sees.

### Compiled Naming in Older Svelte Versions

The exact form of the suffix has changed across major versions. Older Svelte (3.x) emitted `wiggle-abc123` (no `svelte-` prefix); Svelte 5 emits a different style. **Don't pattern-match the suffix** in your source — always write the bare name and let the compiler handle the rewrite.

---

## Global Keyframes (`-global-` prefix)

Prepend the keyframe name with `-global-` to make it available to the entire app. The prefix is stripped at compile time.

```svelte
<style>
  @keyframes -global-fade-in {
    from { opacity: 0; }
    to   { opacity: 1; }
  }
</style>
```

Compiles to:

```css
@keyframes fade-in { from { opacity: 0; } to { opacity: 1; } }
```

The `-global-` part is removed; the rest of the name is preserved verbatim. Other components, third-party scripts, and inline JS can all reference `fade-in` by its bare name.

### Naming Rules

- The prefix is `-global-` (dash, "global", dash).
- No other punctuation: `-global- ` (with a trailing space), `global-` (no leading dash), or `_global_` (underscore) won't be recognized.
- The prefix is case-sensitive: `-Global-` won't be stripped.

### Sharing Across Components

Three reliable strategies:

1. **Top-level shell component with `-global-` keyframes** — render once at app root.
2. **Global stylesheet** — define keyframes in `app.css`, import once.
3. **Pass the keyframe name down via a custom property** — for very dynamic cases.

### Reduced Motion

Pair global keyframes with `prefers-reduced-motion` overrides at the consumer site (where the `animation:` rule lives). The media query is independent of how the `@keyframes` was declared.

---

## {@html} and Scoped Styles (Re-examined)

Recall: `{@html}` content has no scope hash, so scoped selectors don't match. Use `:global(...)` for selectors that need to reach into the injected HTML.

```svelte
<style>
  :global(.highlight) { color: red; }
</style>

<div>{@html trustedHtml}</div>
```

The `:global(...)` rule is emitted as a normal global selector, so it can match the injected `<span class="highlight">` inside the `@html` content.

For slotted content (parent styling the slot), use `::slotted(selector)` from the parent component, not `:global`. `::slotted()` is more restrictive and signals intent.

### Security Warning

Never use `:global(...)` to style HTML you don't fully control. If the content comes from a user, the class names you target can be present in attacker-controlled markup, and the styles will apply. Use a sanitizer or render the content as a separate component with its own scoped styles.

---

## Nested `<style>` Elements (No Scoping)

A Svelte component can have **only one top-level `<style>`**. But you can place additional `<style>` tags *inside* elements, logic blocks, or slots. These nested tags are inserted into the DOM as-is — the compiler does not scope, parse, or rewrite them.

```svelte
<div>
  <style>
    /* This rule applies globally, not just to descendants of this div. */
    div { color: red; }
  </style>
  <p>I'm red.</p>
</div>
```

### Lifecycle

A nested `<style>` is created and destroyed with the element that contains it. This is different from a scoped `<style>`, which is hoisted into a stylesheet at compile time and never destroyed.

```svelte
{#if showStyle}
  <style>
    :root { --theme: dark; }
  </style>
{/if}
```

Toggling `showStyle` adds or removes the `:root` variables — useful for themes that should be entirely absent when inactive.

### In a Loop

```svelte
{#each themes as theme}
  <div>
    <style>
      :root { --color: {theme.color}; }
    </style>
    <span>{theme.name}</span>
  </div>
{/each}
```

Each iteration produces its own `<style>` element. The last one in the DOM wins for equal-specificity rules on `:root` — usually the last item in the array, since it appears later in the DOM.

### In a Slot

```svelte
<!-- Modal.svelte -->
<div class="modal">
  <slot />
</div>
```

```svelte
<!-- App.svelte -->
<Modal>
  <style>
    h2 { color: purple; }
  </style>
  <h2>Title</h2>
</Modal>
```

The `<style>` is rendered as part of the slot content and inserted as-is into the DOM. The `h2` inside the slot is styled, plus any other `h2` on the page.

### Compiled Behaviour

A nested `<style>` produces a real HTMLStyleElement at runtime. The browser parses it like any other `<style>` tag — the Svelte compiler does not touch it. This means:

- `@keyframes` inside a nested style are NOT scoped. They are emitted verbatim.
- The selector `div { color: red }` inside a nested style is NOT scoped. It applies to every `<div>` on the page.
- The compiler does NOT add any scope class to the elements of the parent component. The styles reach wherever the selector reaches.

### When to Use Nested `<style>`

- Toggling global tokens on/off (theme variables that should disappear with the toggle).
- Self-contained third-party integrations (a widget that ships its own CSS).
- One-off dynamic stylesheets that change at runtime.
- Email / SVG content where `<style>` is the standard form.

### When NOT to Use

- For static global styles — use a regular CSS file or `:global { ... }` in a top-level style.
- To add a few extra rules to your component — put them in the top-level `<style>`.
- To style a child component — use a custom property, `:global`, or a real `<style>` in the child.

---

## The Scope Class in the Cascade

When the compiler transforms a selector, it produces a rule whose effective specificity is bumped by `+0-1-0`. This bumps the rule above the same selector in a global stylesheet, regardless of source order.

```svelte
<!-- App.svelte -->
<style>
  p { color: burlywood; }
</style>
```

```css
/* app.css — loaded AFTER App.svelte */
p { color: red; }
```

The `p` in App.svelte's `<style>` is `p.svelte-abc123 { color: burlywood; }` (specificity 0-1-1). The `p` in `app.css` is `p { color: red; }` (specificity 0-0-1). The scoped rule wins.

This is often surprising for new Svelte developers who expect "global" to mean "highest priority." The reverse is true: scoped styles are *more* specific than global ones of the same shape.

### Override Strategies

See [specificity.md](./specificity.md) for the full breakdown. The short version:

1. Increase specificity in the global rule (`html body p { ... }`).
2. Use `:where()` in either rule to lower specificity.
3. Move the scoped rule to a `:global { ... }` block.
4. Use a global stylesheet with `!important` (last resort).

### The `:where()` Wrapping

For deep selectors like `.a .b .c`, the compiler produces:

```css
.a.svelte-xyz :where(.svelte-xyz) .b.svelte-xyz :where(.svelte-xyz) .c.svelte-xyz { ... }
```

The middle occurrences of `.svelte-xyz` are wrapped in `:where()`, which contributes zero specificity. So the *effective* specificity is `0-1-1` (one scope class + one element), not `0-4-1` as it would be without `:where()`. This is what keeps scoped specificity predictable.

---

## Edge Cases and Gotchas

### `:global` inside `:global` is a no-op

```svelte
<style>
  :global { :global { p { color: red; } } }
</style>
```

The inner `:global { ... }` is redundant — the outer one already opts everything inside it out of scoping.

### Pseudo-elements with `:global`

```svelte
<style>
  /* Pseudo-element selectors work normally with scope class. */
  p::first-letter { font-size: 2rem; }
</style>
```

The compiler treats pseudo-elements like part of the compound selector. They get the scope class on the element, and the pseudo-element applies on top.

### `:not()` and `:is()` arguments

The compiler handles nested argument lists by walking each branch and applying the scope class to the most specific one. This usually produces what you'd expect, but watch for surprises with complex `:is()` chains.

```svelte
<style>
  :is(p, .note) { color: blue; }
  /* Compiles to: :is(p.svelte-xyz, .note.svelte-xyz) { color: blue; } */
</style>
```

### Slot children and selector piercing

A child's scoped rule does **not** reach into a parent's slot content. The slot content is in the parent's scope. To style slot children from the parent, use `::slotted()`.

### `class:` vs `class={}` (Svelte 5.16+)

The `class:` directive is older and less flexible. Prefer the object/array form of the `class` attribute in modern Svelte 5.

### Scope hash collisions

In a single Svelte project, scope hashes are derived from the component's contents and path, so two distinct components always get distinct hashes. Two *identical* components in different folders may share a hash — but that's fine because the rules are also identical, so no semantic difference.

---

## See also

- [scoped-keyframes.md](./scoped-keyframes.md) — full reference on `@keyframes` and `-global-`
- [global-reference.md](./global-reference.md) — full reference on `:global(...)`
- [specificity.md](./specificity.md) — full reference on specificity interactions
- [style-directive.md](./style-directive.md) — full reference on the `style:` directive
- [css-custom-properties.md](./css-custom-properties.md) — custom property patterns
- [class-directive.md](./class-directive.md) — class attribute and `class:` directive
