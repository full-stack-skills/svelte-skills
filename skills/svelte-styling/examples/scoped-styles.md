# Scoped Styles Examples

This file covers all scoped styling patterns in Svelte 5.

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
