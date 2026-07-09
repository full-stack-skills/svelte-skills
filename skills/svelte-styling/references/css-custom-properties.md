# CSS Custom Properties in Depth

A comprehensive reference on CSS Custom Properties (CSS Variables) in Svelte 5.

## Passing from Parent to Child via style:--*

Svelte components accept CSS custom properties as attributes on the component element.

### Basic Usage

**Parent.svelte**:
```svelte
<Card
  --card-bg="#f5f5f5"
  --card-border="1px solid #ddd"
  --card-shadow="0 2px 8px rgba(0,0,0,0.1)"
/>
```

**Card.svelte**:
```svelte
<style>
  .card {
    background: var(--card-bg, white);
    border: var(--card-border, none);
    box-shadow: var(--card-shadow, none);
  }
</style>

<div class="card">
  <slot />
</div>
```

### How It Works

1. Parent passes `--variable-name` as an attribute
2. Svelte applies these as inline styles on the child component's root element
3. Child reads them via `var(--variable-name, fallback)`
4. CSS custom properties cascade through the DOM tree

### Multiple Custom Properties

```svelte
<Button
  --btn-bg="#ff3e00"
  --btn-color="white"
  --btn-padding="0.75rem 1.5rem"
  --btn-radius="8px"
  --btn-font-size="1rem"
  --btn-transition="all 0.2s ease"
/>
```

**Button.svelte**:
```svelte
<style>
  .btn {
    background: var(--btn-bg, #ccc);
    color: var(--btn-color, #333);
    padding: var(--btn-padding, 0.5rem 1rem);
    border-radius: var(--btn-radius, 4px);
    font-size: var(--btn-font-size, 1rem);
    transition: var(--btn-transition, none);
  }
</style>

<button class="btn">
  <slot />
</button>
```

---

## Reading with var(--*) Inside <style>

CSS custom properties are read using the `var()` function in CSS declarations.

### Syntax

```css
var(--custom-property-name)
var(--custom-property-name, fallback-value)
```

### Fallback Values

```svelte
<style>
  .element {
    /* No fallback - inherits from parent or initial */
    color: var(--text-color);

    /* With fallback */
    background: var(--card-bg, #ffffff);
    padding: var(--card-padding, 1rem);

    /* Multiple fallbacks (first non-invalid value used) */
    border: var(--card-border, var(--default-border, 1px solid #ccc));
  }
</style>
```

### Multiple Custom Properties

```svelte
<style>
  .card {
    /* Read multiple custom properties */
    background: var(--bg-primary, #fff);
    color: var(--text-primary, #333);
    border: var(--border-color, var(--border-default, #ddd));
    border-radius: var(--radius, 8px);
    padding: var(--spacing, 1rem);
    margin: var(--margin, 0 auto);
  }
</style>
```

---

## Cascading Through Component Tree

CSS custom properties **cascade naturally** through the DOM, making them powerful for theming.

### Theme Example

**App.svelte**:
```svelte
<style>
  :global(:root) {
    --primary: #ff3e00;
    --secondary: #40e0d0;
    --background: #ffffff;
    --text: #333333;
  }
</style>

<div class="app">
  <Header />
  <Main />
  <Footer />
</div>
```

**AnyComponent.svelte**:
```svelte
<style>
  .button {
    background: var(--primary);
    color: var(--background);
  }

  .text {
    color: var(--text);
  }
</style>

<button class="button">Click me</button>
<p class="text">Hello</p>
```

### Nested Theming

```svelte
<!-- DarkTheme.svelte -->
<style>
  .dark-mode {
    --bg: #1a1a1a;
    --text: #f5f5f5;
    --primary: #ff6b35;
  }

  .light-mode {
    --bg: #ffffff;
    --text: #333333;
    --primary: #ff3e00;
  }
</style>

<div class={isDark ? 'dark-mode' : 'light-mode'}>
  <slot />
</div>
```

**Usage**:
```svelte
<DarkTheme>
  <Card /> <!-- Card inherits dark theme variables -->
</DarkTheme>
```

### Override at Any Level

```svelte
<!-- Override at component level -->
<Card --card-bg="#e0f7fa" />

<!-- Override at container level -->
<div style="--card-bg: #e8f5e9; --card-shadow: 0 4px 12px;">
  <Card /> <!-- All Cards here inherit green theme -->
</div>
```

---

## Dynamic CSS Property Updates with $state

Svelte 5's `$state` rune enables reactive CSS custom properties.

### Basic Dynamic Theme

```svelte
<script>
  let isDark = $state(false);

  function toggleTheme() {
    isDark = !isDark;
  }
</script>

<style>
  .container {
    background: var(--bg, white);
    color: var(--text, black);
    min-height: 100vh;
    transition: background 0.3s, color 0.3s;
  }

  :global(:root) {
    --bg: var(--theme-bg, white);
    --text: var(--theme-text, black);
  }
</style>

<div class="container">
  <button onclick={toggleTheme}>
    Toggle to {isDark ? 'Light' : 'Dark'}
  </button>

  <!-- Override CSS vars dynamically -->
  <div
    style={`
      --theme-bg: ${isDark ? '#1a1a1a' : '#ffffff'};
      --theme-text: ${isDark ? '#f5f5f5' : '#333333'};
    `}
  >
    <p>This text changes color dynamically</p>
  </div>
</div>
```

### Interactive Slider with Dynamic CSS Vars

```svelte
<script>
  let hue = $state(200);
  let saturation = $state(80);
  let lightness = $state(50);
</script>

<style>
  .color-preview {
    background: hsl(var(--hue), var(--sat), var(--light));
    width: 200px;
    height: 200px;
    border-radius: 8px;
    transition: background 0.1s;
  }

  .controls {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    max-width: 300px;
  }
</style>

<div class="controls">
  <div class="color-preview" style={`--hue: ${hue}; --sat: ${saturation}%; --light: ${lightness}%;`}></div>

  <label>
    Hue: {hue}
    <input type="range" min="0" max="360" bind:value={hue} />
  </label>

  <label>
    Saturation: {saturation}%
    <input type="range" min="0" max="100" bind:value={saturation} />
  </label>

  <label>
    Lightness: {lightness}%
    <input type="range" min="0" max="100" bind:value={lightness} />
  </label>
</div>
```

### Component Props to CSS Variables

```svelte
<!-- ProgressBar.svelte -->
<script>
  let {
    value = 0,
    --progress-color = '#ff3e00',
    --progress-bg = '#eee',
    --progress-height = '8px',
    ...props
  } = $props();
</script>

<style>
  .track {
    height: var(--progress-height);
    background: var(--progress-bg);
    border-radius: calc(var(--progress-height) / 2);
    overflow: hidden;
  }

  .fill {
    height: 100%;
    background: var(--progress-color);
    width: calc(var(--value) * 1%);
    transition: width 0.3s ease;
  }
</style>

<div class="track" style={`--value: ${value};`} {...props}>
  <div class="fill"></div>
</div>
```

**Usage**:
```svelte
<ProgressBar
  bind:value={progress}
  --progress-color="#22c55e"
  --progress-height="12px"
/>
```

### CSS Variable Binding

```svelte
<script>
  let themeColor = $state('#ff3e00');
</script>

<style>
  .box {
    background: var(--theme-color);
    transition: background 0.3s;
  }
</style>

<input type="color" bind:value={themeColor} />

<div class="box" style={`--theme-color: ${themeColor};`}>
  Color: {themeColor}
</div>
```

---

## Advanced Patterns

### CSS Custom Properties as Bridges

```svelte
<!-- ThemeProvider.svelte -->
<script>
  let { children, theme = 'light' } = $props();

  const themes = {
    light: { --bg: '#fff', --text: '#333' },
    dark: { --bg: '#1a1a1a', --text: '#f5f5f5' },
    blue: { --bg: '#e3f2fd', --text: '#1565c0' },
  };
</script>

<div style={themes[theme]}>
  {@render children()}
</div>
```

### Responsive CSS Variables

```svelte
<style>
  .container {
    --container-padding: 1rem;
  }

  @media (min-width: 768px) {
    .container {
      --container-padding: 2rem;
    }
  }

  @media (min-width: 1024px) {
    .container {
      --container-padding: 3rem;
    }
  }

  .content {
    padding: var(--container-padding);
  }
</style>

<div class="container">
  <div class="content">
    Responsive padding via CSS variable!
  </div>
</div>
```

### Component Variant System

```svelte
<!-- Button.svelte -->
<script>
  let {
    variant = 'primary',
    size = 'medium',
    class: cls = '',
    ...props
  } = $props();

  const variants = {
    primary: { '--btn-bg': '#ff3e00', '--btn-color': 'white' },
    secondary: { '--btn-bg': '#f5f5f5', '--btn-color': '#333' },
    ghost: { '--btn-bg': 'transparent', '--btn-color': '#333' },
  };

  const sizes = {
    small: { '--btn-padding': '0.25rem 0.5rem', '--btn-font': '0.875rem' },
    medium: { '--btn-padding': '0.5rem 1rem', '--btn-font': '1rem' },
    large: { '--btn-padding': '0.75rem 1.5rem', '--btn-font': '1.25rem' },
  };
</script>

<style>
  .btn {
    background: var(--btn-bg);
    color: var(--btn-color);
    padding: var(--btn-padding);
    font-size: var(--btn-font);
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }
</style>

<button
  class={['btn', cls]}
  style={`${variants[variant]}; ${sizes[size]}`}
  {...props}
>
  <slot />
</button>
```

---

## Gotchas and Tips

1. **Undefined is not invalid** - CSS `var()` only falls back for invalid values, not undefined
2. **Whitespace matters** - `var(--color)` and `var(--color, )` behave differently
3. **Type coercion** - CSS vars are strings; cast when using in calculations
4. **Inheritance** - CSS vars inherit through DOM tree, not component tree
5. **Inline styles win** - Component attributes become inline styles, high specificity
