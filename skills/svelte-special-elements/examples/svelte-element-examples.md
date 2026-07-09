# Svelte Special Elements Examples

## 1. svelte:window with Event Listeners

### Keyboard Shortcut Handler

```svelte
<script>
  let pressedKey = $state('');

  function handleKey(event) {
    pressedKey = event.key;
    if (event.key === 's' && (event.metaKey || event.ctrlKey)) {
      event.preventDefault();
      console.log('Save triggered');
    }
  }
</script>

<svelte:window onkeydown={handleKey} />

<p>Last key pressed: {pressedKey}</p>
```

### Responsive Layout with resize

```svelte
<script>
  let width = $state(0);
  let height = $state(0);

  function handleResize() {
    width = window.innerWidth;
    height = window.innerHeight;
  }
</script>

<svelte:window onresize={handleResize} />

<div class="viewport-info">
  <p>Width: {width}px</p>
  <p>Height: {height}px</p>
  {#if width < 768}
    <p>Mobile view</p>
  {:else if width < 1024}
    <p>Tablet view</p>
  {:else}
    <p>Desktop view</p>
  {/if}
</div>
```

## 2. svelte:window with Bindable Properties

```svelte
<script>
  let innerWidth = $state(0);
  let innerHeight = $state(0);
  let scrollX = $state(0);
  let scrollY = $state(0);
  let online = $state(true);

  $effect(() => {
    online = navigator.onLine;
  });
</script>

<svelte:window
  bind:innerWidth
  bind:innerHeight
  bind:scrollX
  bind:scrollY
  bind:online
/>

<div class="info-panel">
  <p>Viewport: {innerWidth} x {innerHeight}</p>
  <p>Scroll: {scrollX}, {scrollY}</p>
  <p>Online: {online ? 'Yes' : 'No'}</p>
</div>
```

## 3. svelte:document with visibilitychange

```svelte
<script>
  let isVisible = $state(true);
  let lastChanged = $state(new Date());

  function handleVisibility() {
    isVisible = document.visibilityState === 'visible';
    lastChanged = new Date();
  }
</script>

<svelte:document onvisibilitychange={handleVisibility} />

<div class="visibility-status">
  <p>Page is {isVisible ? 'visible' : 'hidden'}</p>
  <p>Last changed: {lastChanged.toLocaleTimeString()}</p>
</div>
```

## 4. svelte:document with selectionchange

```svelte
<script>
  let selectedText = $state('');

  function handleSelection() {
    const selection = document.getSelection();
    selectedText = selection ? selection.toString() : '';
  }
</script>

<svelte:document onselectionchange={handleSelection} />

<p>Selected text: "{selectedText}"</p>
```

## 5. svelte:body with mouseenter/mouseleave

```svelte
<script>
  let isHovering = $state(false);
  let mouseX = $state(0);
  let mouseY = $state(0);

  function handleMouseenter() {
    isHovering = true;
  }

  function handleMouseleave() {
    isHovering = false;
  }
</script>

<svelte:body onmouseenter={handleMouseenter} onmouseleave={handleMouseleave} />

<div class="tracking-area" class:hovering={isHovering}>
  <p>Hover status: {isHovering ? 'inside' : 'outside'}</p>
</div>

<style>
  .tracking-area {
    padding: 2rem;
    border: 2px dashed #ccc;
    transition: background 0.3s;
  }
  .tracking-area.hovering {
    background: #e0f7fa;
    border-color: #00bcd4;
  }
</style>
```

## 6. svelte:element Dynamic Tag Switching (h1/h2/h3)

```svelte
<script>
  let level = $state(1);
  let content = $state('Heading Content');

  const levels = [1, 2, 3, 4, 5, 6];
</script>

<div class="controls">
  <label>
    Heading level:
    <select bind:value={level}>
      {#each levels as l}
        <option value={l}>H{l}</option>
      {/each}
    </select>
  </label>
  <input type="text" bind:value={content} placeholder="Heading text" />
</div>

<div class="preview">
  <svelte:element this={'h' + level}>{content}</svelte:element>
  <p>Rendered tag: <code>&lt;h{level}&gt;</code></p>
</div>

<style>
  .controls {
    display: flex;
    gap: 1rem;
    margin-bottom: 1rem;
  }
  .preview {
    padding: 1rem;
    background: #f5f5f5;
    border-radius: 4px;
  }
</style>
```

## 7. svelte:element with Conditional Rendering

```svelte
<script>
  let showElement = $state(true);
  let tag = $state('div');
</script>

<label>
  <input type="checkbox" bind:checked={showElement} />
  Show element
</label>

<label>
  Tag:
  <select bind:value={tag}>
    <option value="div">div</option>
    <option value="article">article</option>
    <option value="section">section</option>
    <option value="aside">aside</option>
  </select>
</label>

{#if showElement}
  <svelte:element this={tag} class="conditional-box">
    Conditional content in a {tag}
  </svelte:element>
{/if}

<style>
  .conditional-box {
    padding: 1rem;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border-radius: 8px;
    margin-top: 1rem;
  }
</style>
```

## 8. svelte:element with SVG Namespace

```svelte
<script>
  let svgTag = $state('circle');
</script>

<div class="svg-controls">
  <label>
    SVG element:
    <select bind:value={svgTag}>
      <option value="circle">circle</option>
      <option value="rect">rect</option>
      <option value="line">line</option>
      <option value="ellipse">ellipse</option>
    </select>
  </label>
</div>

<svg viewBox="0 0 200 200" xmlns="http://www.w3.org/2000/svg">
  <svelte:element
    this={svgTag}
    cx="100"
    cy="100"
    r="80"
    fill="#667eea"
    stroke="#764ba2"
    stroke-width="3"
  />
</svg>
```

## 9. svelte:component (Svelte 4 Style Dynamic Component)

```svelte
<script>
  import Header from './Header.svelte';
  import Footer from './Footer.svelte';
  import Article from './Article.svelte';

  let currentComponent = $state('header');

  const components = {
    header: Header,
    article: Article,
    footer: Footer
  };
</script>

<div class="component-switcher">
  <button onclick={() => currentComponent = 'header'}>Header</button>
  <button onclick={() => currentComponent = 'article'}>Article</button>
  <button onclick={() => currentComponent = 'footer'}>Footer</button>
</div>

<div class="component-container">
  <svelte:component this={components[currentComponent]} />
</div>

<style>
  .component-switcher {
    display: flex;
    gap: 0.5rem;
    margin-bottom: 1rem;
  }
  .component-container {
    padding: 1rem;
    background: #fafafa;
    border: 1px solid #ddd;
    border-radius: 4px;
  }
</style>
```

## 10. svelte:self for Recursive Components (Old Svelte 4 Way)

```svelte
<!-- TreeNode.svelte -->
<script>
  let { node, depth = 0 } = $props();
</script>

<div class="tree-node" style="padding-left: {depth * 20}px">
  <svelte:self node={node.children} depth={depth + 1} />
</div>

<!-- Note: In Svelte 5, recursive components typically use the component name directly -->
<!-- <TreeNode /> instead of <svelte:self /> in most cases -->
```

## 11. svelte:options customElement with $host

```svelte
<!-- Card.svelte -->
<svelte:options customElement="my-card" />

<script>
  let { title = 'Default Title', children } = $props();
</script>

<host>
  <div class="card">
    <h2>{title}</h2>
    <div class="content">
      {@render children?.()}
    </div>
  </div>
</host>

<style>
  .card {
    border: 1px solid #ddd;
    border-radius: 8px;
    padding: 1rem;
    background: white;
  }
  h2 {
    margin: 0 0 1rem 0;
    font-size: 1.25rem;
  }
  .content {
    color: #666;
  }
</style>
```

## 12. svelte:options runes={false} for Legacy Mode

```svelte
<!-- LegacyComponent.svelte -->
<svelte:options runes={false} />

<script>
  // Svelte 4 style reactive declarations still work
  let count = 0;
  $: doubled = count * 2;
  $: if (count > 10) console.log('Count exceeded 10');

  function increment() {
    count += 1;
  }
</script>

<button onclick={increment}>
  Count: {count} (doubled: {doubled})
</button>
```

## 13. svelte:options namespace="svg"

```svelte
<!-- Icon.svelte -->
<svelte:options namespace="svg" />

<script>
  let { size = 24, color = 'currentColor' } = $props();
</script>

<g fill={color}>
  <circle cx={size / 4} cy={size / 2} r={size / 8} />
  <circle cx={size * 3 / 4} cy={size / 2} r={size / 8} />
  <path d="M {size / 4} {size * 3 / 4} Q {size / 2} {size / 4} {size * 3 / 4} {size * 3 / 4}" />
</g>

<!-- Usage in an SVG context: -->
<!-- <svg><Icon size={32} color="blue" /></svg> -->
```

## 14. svelte:options with Multiple Settings

```svelte
<!-- AdvancedSettings.svelte -->
<svelte:options
  customElement="advanced-element"
  namespace="svg"
  runes={true}
/>

<script>
  // Component code here
</script>

<g class="advanced-element">
  <rect x="0" y="0" width="100" height="100" fill="#667eea" />
</g>
```

## Summary of Key Patterns

| Element | Use Case | Key Feature |
|---------|----------|-------------|
| `svelte:window` | Global keyboard, resize, scroll | bindable properties |
| `svelte:document` | visibilitychange, selectionchange | readonly bindings |
| `svelte:body` | mouseenter, mouseleave | body-level events |
| `svelte:element` | Dynamic HTML tags | `this` attribute |
| `svelte:component` | Dynamic Svelte components | `this` prop |
| `svelte:self` | Recursive components (legacy) | Self-reference |
| `svelte:options` | Compiler configuration | Various options |
