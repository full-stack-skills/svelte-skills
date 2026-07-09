# class vs class: Directive

A comprehensive reference on Svelte's class attribute and class: directive patterns.

## Object Form class={{ }}

Use an object to conditionally apply class names (clsx-style pattern).

### Syntax

```svelte
<div class={{
  className: condition,
  anotherClass: anotherCondition,
}}>
  Content
</div>
```

- Keys are class names
- Values are truthy/falsy
- Truthy values add the class name

### Basic Example

```svelte
<script>
  let isActive = $state(false);
  let isLarge = $state(true);
</script>

<style>
  .badge {
    display: inline-flex;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
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

<div class={{ active: isActive, large: isLarge }}>
  Badge
</div>
```

**Result when `isActive=false`, `isLarge=true`**:
```html
<div class="large">Badge</div>
```

**Result when `isActive=true`, `isLarge=true`**:
```html
<div class="active large">Badge</div>
```

---

## Array Form class={[ ]}

Use an array to combine multiple class sources.

### Syntax

```svelte
<div class={[
  'base-class',
  condition && 'conditional-class',
  anotherCondition ? 'class-a' : 'class-b',
  props.class
]}>
  Content
</div>
```

- Arrays are flattened
- Falsy values are skipped
- Multiple sources can be combined

### Basic Example

```svelte
<script>
  let isFaded = $state(true);
  let isUnderline = $state(false);
</script>

<style>
  .text {
    font-size: 1rem;
  }

  .faded {
    opacity: 0.5;
  }

  .underline {
    text-decoration: underline;
  }

  .highlight {
    background: yellow;
  }
</style>

<div class={[
  'text',
  isFaded && 'faded',
  isUnderline && 'underline',
  'highlight'
]}>
  Array class example
</div>
```

### Combining Static and Dynamic

```svelte
<Button class={['btn', isPrimary && 'btn-primary', isLoading && 'btn-loading']} />
```

### Spreading Props

```svelte
<script>
  let { class: cls = '', ...props } = $props();
</script>

<button class={['base-btn', cls]}>
  <slot />
</button>
```

---

## Shorthand class: Directive (Svelte 5.16+)

The `class:` directive provides a shorthand for boolean class toggling.

### Syntax

```svelte
<div class:active class:large={isLarge}>
  Content
</div>
```

### Equivalence

```svelte
<!-- class: directive -->
<div class:active class:large={isLarge}>

<!-- Equivalent class object -->
<div class={{ active: active, large: isLarge }}>
```

### Example

```svelte
<script>
  let isActive = $state(true);
  let isDisabled = $state(false);
</script>

<style>
  .tab {
    padding: 0.5rem 1rem;
    border: 1px solid #ddd;
    cursor: pointer;
  }

  .active {
    background: #ff3e00;
    color: white;
    border-color: #ff3e00;
  }

  .disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
</style>

<div class="tab" class:active={isActive} class:disabled={isDisabled}>
  Tab Content
</div>
```

---

## clsx Behavior

The object form mimics `clsx`/`tailwind-merge` behavior.

### Truthy Values

```svelte
<!-- truthy values add class -->
<div class={{ active: true, disabled: 1, hidden: 'yes' }}>
  <!-- classes: "active disabled hidden" -->
</div>

<!-- falsy values don't add class -->
<div class={{ active: false, disabled: 0, hidden: null, undefined: undefined }}>
  <!-- classes: "" -->
</div>
```

### String Values

```svelte
<!-- strings are always truthy -->
<div class={{ active: 'is-active', disabled: 'is-disabled' }}>
  <!-- classes: "is-active is-disabled" -->
</div>
```

### Number Values

```svelte
<!-- numbers are truthy when non-zero -->
<div class={{ count: 0, index: 1 }}>
  <!-- classes: "index" (0 is falsy, 1 is truthy) -->
</div>
```

---

## ClassValue Type for TypeScript

Use `svelte/elements` types for type-safe class props.

### ClassValue Type

```typescript
import type { ClassValue } from 'svelte/elements';

// Accept various class value types
type ClassValue = string | number | boolean | undefined | null | ClassValue[] | { [key: string]: unknown };
```

### Basic Type-Safe Class Prop

```svelte
<script lang="ts">
  import type { ClassValue } from 'svelte/elements';

  let {
    class: cls = '',
    ...props
  }: { class?: ClassValue } = $props();
</script>

<button class={['btn', cls]}>
  <slot />
</button>
```

### Merging Classes

```svelte
<script lang="ts">
  import type { ClassValue } from 'svelte/elements';

  let {
    variant = 'primary',
    class: cls = '',
    ...props
  }: {
    variant?: 'primary' | 'secondary' | 'ghost';
    class?: ClassValue;
  } = $props();

  const variantClasses = {
    primary: 'bg-blue-500 text-white',
    secondary: 'bg-gray-200 text-gray-800',
    ghost: 'bg-transparent hover:bg-gray-100',
  };
</script>

<button class={[variantClasses[variant], cls]} {...props}>
  <slot />
</button>
```

### Utility Type Helper

```typescript
// type utility for class props
type ClassProp<T extends string = string> = {
  class?: ClassValue;
};

// Component using the helper
interface ButtonProps extends ClassProp {
  variant?: 'primary' | 'secondary';
  disabled?: boolean;
}
```

---

## Comparison Table

| Feature | `class={{ }}` | `class={[ ]}` | `class:name` |
|---------|---------------|---------------|--------------|
| Conditional | Yes (object values) | Yes (truthy check) | Yes (boolean) |
| Multiple classes | Yes | Yes | No |
| Dynamic name | No | No | No |
| Readability | Good for many conditions | Good for lists | Good for single toggle |
| Svelte 5.16+ recommended | Yes | Yes | Deprecated |

---

## Migration from class: to class=

### Before (class: directive)

```svelte
<script>
  let isActive = $state(false);
  let isLarge = $state(false);
</script>

<div class:active={isActive} class:large={isLarge}>
  Content
</div>
```

### After (class object)

```svelte
<script>
  let isActive = $state(false);
  let isLarge = $state(false);
</script>

<div class={{ active: isActive, large: isLarge }}>
  Content
</div>
```

### Array Form Equivalents

```svelte
<!-- Before -->
<div class:active={isActive} class="base-class">

<!-- After -->
<div class={['base-class', { active: isActive }]}>
```

---

## Best Practices

1. **Use `class={}` object/array form** - It's the recommended approach in Svelte 5.16+
2. **Create reusable class utilities** - Factor out common class combinations
3. **Use TypeScript ClassValue** - For type-safe class props
4. **Combine with CSS custom properties** - For variant theming
5. **Avoid class: directive for new code** - It's deprecated in favor of `class={}`

### Example: Reusable Button Component

```svelte
<script lang="ts">
  import type { ClassValue } from 'svelte/elements';

  let {
    variant = 'primary',
    size = 'medium',
    disabled = false,
    loading = false,
    class: cls = '',
    ...props
  }: {
    variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
    size?: 'sm' | 'md' | 'lg';
    disabled?: boolean;
    loading?: boolean;
    class?: ClassValue;
  } = $props();

  const classes = [
    'btn',
    `btn-${variant}`,
    `btn-${size}`,
    loading && 'btn-loading',
    disabled && 'btn-disabled',
    cls
  ];
</script>

<style>
  .btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-weight: 500;
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.2s;
  }

  .btn-primary { background: #ff3e00; color: white; }
  .btn-secondary { background: #f5f5f5; color: #333; }
  .btn-ghost { background: transparent; }
  .btn-danger { background: #dc2626; color: white; }

  .btn-sm { padding: 0.25rem 0.5rem; font-size: 0.875rem; }
  .btn-md { padding: 0.5rem 1rem; font-size: 1rem; }
  .btn-lg { padding: 0.75rem 1.5rem; font-size: 1.125rem; }

  .btn-disabled, .btn-loading { opacity: 0.5; cursor: not-allowed; }
</style>

<button
  class={classes}
  disabled={disabled || loading}
  {...props}
>
  {#if loading}
    <span>Loading...</span>
  {:else}
    <slot />
  {/if}
</button>
```
