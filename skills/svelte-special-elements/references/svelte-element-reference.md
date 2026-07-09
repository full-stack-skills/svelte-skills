# svelte:element Reference

Complete reference for all `svelte:` elements that render content or handle events.

## svelte:window

Listens to window events and binds to window properties.

### Syntax

```svelte
<svelte:window
  onkeydown={handler}
  onresize={handler}
  onscroll={handler}
  bind:innerWidth
  bind:innerHeight
  bind:outerWidth
  bind:outerHeight
  bind:scrollX
  bind:scrollY
  bind:online
/>
```

### Bindable Properties

| Property | Type | Description |
|----------|------|-------------|
| `innerWidth` | `number` | Window viewport width |
| `innerHeight` | `number` | Window viewport height |
| `outerWidth` | `number` | Window outer width (includes chrome) |
| `outerHeight` | `number` | Window outer height (includes chrome) |
| `scrollX` | `number` | Horizontal scroll position |
| `scrollY` | `number` | Vertical scroll position |
| `online` | `boolean` | Network connectivity status |

### Supported Events

All window-level events are supported:

- `onkeydown`, `onkeyup`, `onkeypress`
- `onresize`, `onscroll`
- `onclick`, `oncontextmenu`, `ondblclick`
- `onmousedown`, `onmousemove`, `onmouseup`
- `onmousewheel`, `onwheel`
- `ontouchstart`, `ontouchmove`, `ontouchend`
- `onbeforeinput`, `oninput`, `onchange`
- `ondrag`, `ondragstart`, `ondragend`, `ondragover`
- `ondrop`, `onfocus`, `onblur`
- `oncopy`, `oncut`, `onpaste`
- `onpointerdown`, `onpointermove`, `onpointerup`
- `onpointercancel`, `ongotpointercapture`, `onlostpointercapture`

### Constraints

- Can only appear at the **top level** of a component
- Cannot be inside `{#if}` blocks, `{#each}` blocks, or other control flow
- Cannot be conditionally rendered

---

## svelte:document

Listens to document-level events not available on window.

### Syntax

```svelte
<svelte:document
  onvisibilitychange={handler}
  onselectionchange={handler}
  onreadystatechange={handler}
  bind:visibilityState
  bind:activeElement
  bind:fullscreenElement
  bind:pointerLockElement
/>
```

### Bindable Properties (readonly)

| Property | Type | Description |
|----------|------|-------------|
| `visibilityState` | `'visible' \| 'hidden'` | Document visibility |
| `activeElement` | `Element \| null` | Currently focused element |
| `fullscreenElement` | `Element \| null` | Element in fullscreen mode |
| `pointerLockElement` | `Element \| null` | Element with pointer lock |

### Supported Events

- `onvisibilitychange` - Tab becomes visible/hidden
- `onselectionchange` - User changes text selection
- `onreadystatechange` - Document ready state changes
- `onDOMContentLoaded` - DOM is ready

### Use Cases

Use `svelte:document` when you need events that only fire on `document`, not `window`:

```svelte
<svelte:document onvisibilitychange={handleVisibility} />
```

---

## svelte:body

Listens to body-level events that do not fire on window.

### Syntax

```svelte
<svelte:body
  onmouseenter={handler}
  onmouseleave={handler}
  onload={handler}
/>
```

### Supported Events

- `onmouseenter` - Mouse enters the document body
- `onmouseleave` - Mouse leaves the document body
- `onload` - Body has loaded

### Why svelte:body Exists

Some mouse events do not propagate to `window`. The `mouseenter` and `mouseleave` events specifically do not fire on `window`, making `svelte:body` necessary:

```svelte
<svelte:body onmouseenter={handleMouseEnter} onmouseleave={handleMouseLeave} />
```

---

## svelte:element

Renders a dynamic HTML element where the tag name is determined at runtime.

### Syntax

```svelte
<svelte:element this={tagName}>content</svelte:element>
<svelte:element this={tagName} attribute={value} />
<svelte:element this={tagName} xmlns="http://www.w3.org/2000/svg" />
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `this` | `string \| null \| void` | The HTML tag to render |

### Behavior

- If `this` is `null` or `undefined`, nothing renders
- If `this` is a void element (`<br>`, `<hr>`, `<img>`, etc.) but has children, a **runtime error** is thrown
- Svelte automatically infers namespace (`svg`, `mathml`) from context
- Use `xmlns` attribute to explicitly set namespace

### Supported Bindings

Only `bind:this` is supported on `svelte:element` because HTML elements do not have Svelte-specific bindings like `bind:value`:

```svelte
<script>
  let el;
  let tag = 'div';
</script>

<svelte:element this={tag} bind:this={el}>
  <button onclick={() => el.remove()}>Remove</button>
</svelte:element>
```

### Examples

**Dynamic heading levels:**
```svelte
<script>
  let level = 1;
</script>

<svelte:element this={'h' + level}>Heading</svelte:element>
```

**SVG elements:**
```svelte
<svelte:element this="circle" cx="50" cy="50" r="40" />
```

---

## svelte:component

Renders a dynamic Svelte component based on a component constructor.

### Syntax

```svelte
<svelte:component this={component} prop={value} />
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `this` | `Component \| null \| void` | The component constructor to render |

### Behavior

- If `this` is `null` or `undefined`, nothing renders
- When `this` changes, the component is destroyed and recreated
- Component receives all other attributes as props

### Example

```svelte
<script>
  import RedBox from './RedBox.svelte';
  import BlueBox from './BlueBox.svelte';

  let current = $state(RedBox);
</script>

<button onclick={() => current = current === RedBox ? BlueBox : RedBox}>
  Toggle
</button>

<svelte:component this={current} />
```

### In Svelte 5

While `svelte:component` still works in Svelte 5 for backward compatibility, the preferred approach is to use snippets or render functions:

```svelte
<script>
  import RedBox from './RedBox.svelte';
  import BlueBox from './BlueBox.svelte';

  let current = $state('red');

  const components = {
    red: RedBox,
    blue: BlueBox
  };
</script>

<!-- Preferred in Svelte 5 -->
<DynamicRenderer>
  {#snippet children()}
    {@const Component = components[current]}
    <Component />
  {/snippet}
</DynamicRenderer>
```

---

## svelte:head

Injects content into `document.head`.

### Syntax

```svelte
<svelte:head>
  <title>Page Title</title>
  <meta name="description" content="Description" />
  <link rel="stylesheet" href="styles.css" />
</svelte:head>
```

### SSR Behavior

In server-side rendering, `svelte:head` content is exposed separately and not included in the body HTML. This is useful for SEO.

---

## svelte:boundary

Captures errors and pending states in a portion of the component tree (Svelte 5.3+).

### Syntax

```svelte
<svelte:boundary onerror={handler}>
  <Component />

  {#snippet failed(error)}
    <p>Error: {error.message}</p>
  {/snippet}

  {#snippet pending()}
    <p>Loading...</p>
  {/snippet}
</svelte:boundary>
```

### What is Caught

- Synchronous errors during rendering
- Async errors from Promises
- Pending states from `{#await}` blocks

### What is NOT Caught

- Errors inside event handlers
- Errors inside `setTimeout`, `setInterval`
- Errors in code outside the boundary

---

## svelte:self

Recursively renders the current component (Svelte 4 legacy).

### Syntax

```svelte
<svelte:self />
<svelte:self prop={value} />
```

### Use Case

Used for recursive component trees like file systems or comment threads:

```svelte
<!-- TreeNode.svelte -->
<script>
  let { children = [], name } = $props();
</script>

<div class="tree-node">
  <span>{name}</span>
  {#each children as child}
    <svelte:self {...child} />
  {/each}
</div>
```

### In Svelte 5

The component can reference itself by name directly:

```svelte
<!-- TreeNode.svelte -->
<script>
  let { children = [], name } = $props();
</script>

<div class="tree-node">
  <span>{name}</span>
  {#each children as child}
    <TreeNode {...child} />
  {/each}
</div>
```

---

## svelte:fragment

Defines fallback content for named slots (Svelte 5).

### Syntax

```svelte
<slot name="header" />

<!-- Fallback content -->
<svelte:fragment slot="header">
  <h1>Default Header</h1>
</svelte:fragment>
```

---

## Comparison: svelte:element vs svelte:component

| Aspect | `svelte:element` | `svelte:component` |
|--------|------------------|-------------------|
| Renders | HTML element | Svelte component |
| `this` type | `string` tag name | Component constructor |
| Bindings | `bind:this` only | All Svelte bindings |
| Use case | Dynamic HTML (CMS, data-driven) | Dynamic components |
| Namespace | Supports `xmlns` | No namespace needed |
| Performance | Lightweight | Component lifecycle |

### When to Use Each

**Use `svelte:element` when:**
- Building a CMS that stores HTML tags in a database
- Rendering user-generated content with specific tags
- Working with SVG elements dynamically

**Use `svelte:component` when:**
- Switching between different Svelte components
- Components have different props and behavior
- Need full Svelte component features (bindings, reactivity)
