# svelte:options Reference

Complete reference for the `<svelte:options>` compiler configuration element.

## Syntax

```svelte
<svelte:options option1={value1} option2={value2} />
```

`<svelte:options>` must appear at the top of the component file, after any `<script>` tags.

---

## Option: `runes`

Enables or disables Runes Mode for the component.

### Values

| Value | Behavior |
|-------|----------|
| `true` | Forces Runes Mode (`$state`, `$derived`, `$effect`, etc.) |
| `false` | Forces Legacy Mode (Svelte 4 reactive declarations) |

### Example

```svelte
<svelte:options runes={true} />

<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
</script>

<button onclick={() => count++}>{count} x 2 = {doubled}</button>
```

### Legacy Mode Example

```svelte
<svelte:options runes={false} />

<script>
  let count = 0;
  $: doubled = count * 2;
  $: if (count > 5) console.log('count exceeded 5');
</script>

<button on:click={() => count++}>{count} x 2 = {doubled}</button>
```

### When to Use

- `runes={true}`: When using Svelte 5 features or new components
- `runes={false}`: When maintaining Svelte 4 components or libraries
- Omit entirely: Inherit mode from parent or compiler settings

---

## Option: `namespace`

Sets the XML namespace for the component, affecting how elements are parsed.

### Values

| Value | Namespace | Use Case |
|-------|-----------|----------|
| `"html"` | XHTML | Default, standard HTML |
| `"svg"` | SVG | SVG graphics |
| `"mathml"` | MathML | Mathematical formulas |

### Example: SVG Namespace

```svelte
<svelte:options namespace="svg" />

<script>
  let { size = 24 } = $props();
</script>

<circle cx={size / 2} cy={size / 2} r={size / 3} fill="currentColor" />
```

**Usage:**
```svelte
<svg>
  <Icon size={32} color="blue" />
</svg>
```

### When to Use

Use `namespace="svg"` when the component:
- Will be used as a child inside `<svg>` elements
- Contains SVG elements that should be part of the SVG DOM
- Needs proper SVG element inheritance (styles, transforms, etc.)

### Common Components with SVG Namespace

- Icons
- Charts
- Vector graphics
- Custom SVG elements

---

## Option: `customElement`

Compiles the component as a custom element (Web Component).

### Values

| Value | Behavior |
|-------|----------|
| `"tag-name"` | Custom element tag name (kebab-case required) |
| `"my-element"` | Registers as `<my-element>` |
| `"my-custom-element"` | Registers as `<my-custom-element>` |

### Example

```svelte
<svelte:options customElement="my-button" />

<script>
  let { variant = 'primary', children } = $props();
</script>

<button class={variant}>
  {@render children?.()}
</button>

<style>
  button {
    padding: 0.5rem 1rem;
    border-radius: 4px;
    border: none;
    cursor: pointer;
  }
  .primary {
    background: #667eea;
    color: white;
  }
  .secondary {
    background: #e0e0e0;
    color: #333;
  }
</style>
```

### Usage

```html
<script type="module" src="my-button.js"></script>
<my-button variant="secondary">Click me</my-button>
```

### Custom Element Options

```svelte
<svelte:options
  customElement={{
    tag: 'my-element',
    props: {
      name: { attribute: 'name', reflect: true },
      value: { attribute: 'value', reflect: true }
    },
    observedAttributes: ['name', 'value']
  }}
/>
```

---

## Option: `css`

Controls how component styles are handled.

### Values

| Value | Behavior | Use Case |
|-------|----------|----------|
| `"injected"` | Styles injected via JavaScript at runtime | CSR, dynamic styles |
| `"external"` | Styles in separate CSS file (default SSR) | SSR with external CSS |
| omitted | Default behavior based on compiler mode | Standard usage |

### Example

```svelte
<svelte:options css="injected" />

<div class="styled">This component's styles are injected</div>

<style>
  .styled {
    color: #667eea;
    font-weight: bold;
  }
</style>
```

### When to Use `css="injected"`

- Client-side rendering with scoped styles
- When you need styles to be dynamic
- Shadow DOM scenarios where external CSS doesn't reach

---

## Option: `preserveComments`

Preserves HTML comments in the compiled output.

### Values

| Value | Behavior |
|-------|----------|
| `true` | HTML comments are kept |
| `false` | HTML comments are removed (default) |

### Example

```svelte
<svelte:options preserveComments={true} />

<!-- This comment will be in the output -->
<div>Content</div>
```

---

## Option: `preserveWhitespace`

Preserves whitespace in the compiled output.

### Values

| Value | Behavior |
|-------|----------|
| `true` | Whitespace is preserved |
| `false` | Whitespace is collapsed/minified (default) |

### When to Use

- Preformatted text components
- Markdown renderers
- Code display components

---

## Option: `compileOptions`

Passes additional compiler options for this component.

### Syntax

```svelte
<svelte:options compileOptions={{ option: value }} />
```

### Available Options

```svelte
<svelte:options compileOptions={{
  runes: true,
  dev: true,
  css: 'injected'
}} />
```

### Nested Options

```svelte
<svelte:options compileOptions={{
  generate: 'client',
  filename: 'Custom.svelte',
  sourcemap: true
}} />
```

---

## Option: `accessors`

Generates getters and setters for component props.

### Values

| Value | Behavior |
|-------|----------|
| `true` | Props have getter/setter accessors |
| `false` | No accessors generated (default) |

### Example

```svelte
<svelte:options accessors={true} />

<script>
  let { value = 0 } = $props();
</script>

<input bind:value />
```

**Usage:**
```svelte
<script>
  import MyInput from './MyInput.svelte';
  let input;
</script>

<MyInput bind:value={val} />
```

---

## Option: `immutable`

Optimizes reactivity for immutable data structures.

### Values

| Value | Behavior |
|-------|----------|
| `true` | Props are treated as immutable |
| `false` | Props are compared by reference (default) |

### When to Use

With libraries like Immer or when using persistent data structures:

```svelte
<svelte:options immutable={true} />
```

---

## Multiple Options

Combine multiple options in a single `<svelte:options>` element:

```svelte
<svelte:options
  runes={true}
  namespace="svg"
  customElement="my-icon"
  css="injected"
  preserveComments={false}
  preserveWhitespace={false}
/>
```

---

## Option: `tag`

Specifies the custom element tag name (alternative to `customElement`).

### Syntax

```svelte
<svelte:options tag="my-element" />
```

### Equivalent To

```svelte
<svelte:options customElement="my-element" />
```

---

## Option: `向外`

Exposes props to custom elements.

### Syntax

```svelte
<svelte:options
  customElement="my-element"
  props={{
    name: { attribute: 'name' },
    value: { attribute: 'value', type: 'Number' }
  }}
/>
```

---

## Complete Options Table

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `runes` | `boolean` | inherited | Enable/disable Runes Mode |
| `namespace` | `string` | `"html"` | XML namespace (`html`, `svg`, `mathml`) |
| `customElement` | `string \| object` | none | Compile as custom element |
| `css` | `string` | varies | CSS handling (`injected`, `external`) |
| `preserveComments` | `boolean` | `false` | Keep HTML comments |
| `preserveWhitespace` | `boolean` | `false` | Keep whitespace |
| `accessors` | `boolean` | `false` | Generate prop accessors |
| `immutable` | `boolean` | `false` | Treat props as immutable |
| `tag` | `string` | none | Custom element tag name |
| `compileOptions` | `object` | `{}` | Nested compiler options |

---

## Platform-Specific Options

### Client-Side Rendering

```svelte
<svelte:options compileOptions={{ generate: 'client' }} />
```

### Server-Side Rendering

```svelte
<svelte:options compileOptions={{ generate: 'server' }} />
```

### Hydration

```svelte
<svelte:options compileOptions={{ generate: 'ssr', hydration: true }} />
```

---

## Legacy Options

These options exist for backward compatibility:

### `legacy`

```svelte
<svelte:options legacy={{ reactive: true }} />
```

### `dev`

```svelte
<svelte:options compileOptions={{ dev: true }} />
```

---

## TypeScript Integration

When using TypeScript with custom elements:

```svelte
<svelte:options
  customElement={{
    tag: 'my-element',
    props: {
      count: { type: 'Number', attribute: 'count' },
      name: { type: 'String', attribute: 'name' }
    }
  }}
/>
```
