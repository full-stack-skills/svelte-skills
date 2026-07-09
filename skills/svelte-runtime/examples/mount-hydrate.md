# Mount, Unmount, and Hydrate Examples

Demonstrates Svelte 5's imperative component APIs for client-side mounting, hydration, and lifecycle management.

## 1. Basic mount() Usage

Mount a Svelte component to an existing DOM element:

```js
// main.js
import { mount } from 'svelte';
import App from './App.svelte';

const app = mount(App, {
  target: document.querySelector('#app')
});
```

```svelte
<!-- App.svelte -->
<script>
  let count = $state(0);
</script>

<button onclick={() => count++}>
  Count: {count}
</button>
```

**Output HTML:**
```html
<div id="app">
  <button>Count: 0</button>
</div>
```

## 2. Mount with Props

Pass initial data to the component:

```js
import { mount } from 'svelte';
import UserCard from './UserCard.svelte';

const app = mount(UserCard, {
  target: document.querySelector('#user'),
  props: {
    name: 'Ada Lovelace',
    role: 'Mathematician',
    active: true
  }
});
```

```svelte
<!-- UserCard.svelte -->
<script>
  let { name, role, active } = $props();
</script>

<div class:active>
  <h2>{name}</h2>
  <p>{role}</p>
</div>

<style>
  .active { border-color: green; }
</style>
```

## 3. unmount() with Outro

Unmount a component after playing outro animations:

```js
import { mount, unmount } from 'svelte';
import Modal from './Modal.svelte';

const modal = mount(Modal, {
  target: document.body,
  props: { open: true }
});

// Close button handler
function closeModal() {
  // { outro: true } waits for CSS transitions to complete
  unmount(modal, { outro: true });
}
```

```svelte
<!-- Modal.svelte -->
<script>
  let { open } = $props();
</script>

{#if open}
  <div class="modal-backdrop" transition:fade>
    <div class="modal" transition:fly={{ y: -50 }}>
      <slot />
      <button onclick={() => open = false}>Close</button>
    </div>
  </div>
{/if}

<script context="module">
  import { fade, fly } from 'svelte/transition';
</script>
```

The `outro: true` option ensures:
1. Component stays in DOM while transitions play
2. All `transition:` and `animate:` animations complete
3. Then the DOM element is removed

## 4. hydrate() for SSR Hydration

Reuse server-rendered HTML and make it interactive:

**Server (SvelteKit):**
```js
// +page.server.js
import { render } from 'svelte/server';

export async function load({ fetch }) {
  const data = await fetch('/api/user').then(r => r.json());
  const { head, body } = await render(UserPage, { props: { user: data } });

  return { head, body, user: data }; // user serializable for hydration
}
```

```svelte
<!-- +page.svelte -->
<script>
  let { data } = $props();
  // data.user is available client-side without re-fetching
</script>

{@html data.body}
```

**Client hydration:**
```js
// +page.js or +layout.js
import { hydrate } from 'svelte';
import UserPage from './UserPage.svelte';

const app = hydrate(UserPage, {
  target: document.querySelector('#app'),
  props: { user: window.__INITIAL_DATA__.user }
});
```

**Key difference from mount():**
- `mount()` replaces all target content
- `hydrate()` preserves existing HTML and attaches event listeners

## 5. render() for Server-Side Rendering

Generate HTML on the server:

```js
// server-render.js
import { render } from 'svelte/server';
import App from './App.svelte';

export async function renderApp(props) {
  const { head, body } = await render(App, {
    props
  });

  return `
    <!DOCTYPE html>
    <html>
      <head>${head}</head>
      <body>
        <div id="app">${body}</div>
      </body>
    </html>
  `;
}
```

```svelte
<!-- App.svelte -->
<script>
  let { title = 'Welcome', items = [] } = $props();
</script>

<h1>{title}</h1>
<ul>
  {#each items as item}
    <li>{item}</li>
  {/each}
</ul>
```

## 6. flushSync() to Force Sync Updates

In tests or when you need immediate DOM updates:

```js
import { mount, unmount, flushSync } from 'svelte';
import Counter from './Counter.svelte';

test('counter increments immediately', () => {
  const target = document.createElement('div');
  const counter = mount(Counter, { target, props: { initial: 0 } });

  // Click and wait for sync update
  target.querySelector('button').click();
  flushSync(); // Force all pending effects to run

  expect(target.innerHTML).toContain('Count: 1');

  unmount(counter);
});
```

```svelte
<!-- Counter.svelte -->
<script>
  let { initial = 0 } = $props();
  let count = $state(initial);
</script>

<button onclick={() => count++}>
  Count: {count}
</button>
```

**Why flushSync matters:**

Without it, the click handler updates `count`, but effects/deferred updates haven't run yet. `flushSync()` forces the Svelte runtime to process all pending updates synchronously.

## 7. tick() for Async DOM Updates

Wait for Svelte to finish updating the DOM:

```js
import { mount } from 'svelte';
import AutoResize from './AutoResize.svelte';

const app = mount(AutoResize, { target: document.querySelector('#editor') });

// Programmatically change content and wait for render
app.dispatchEvent(new CustomEvent('set-content', {
  detail: { text: 'New long content that might overflow' }
}));

await tick(); // DOM is now updated

const height = document.querySelector('#editor').scrollHeight;
```

```svelte
<!-- AutoResize.svelte -->
<script>
  import { tick } from 'svelte';

  let content = $state('');
  let textarea;

  $effect(() => {
    // React to content changes
    if (textarea) {
      textarea.style.height = 'auto';
      textarea.style.height = textarea.scrollHeight + 'px';
    }
  });

  function handleSetContent(e) {
    content = e.detail.text;
  }
</script>

<textarea
  bind:this={textarea}
  onsetcontent={handleSetContent}
  value={content}
></textarea>
```

## 8. Multiple mount() Calls on Same Page

Mount multiple independent component instances:

```js
import { mount } from 'svelte';
import TodoItem from './TodoItem.svelte';

const container = document.querySelector('#todos');

const items = [
  { id: 1, text: 'Learn Svelte', done: false },
  { id: 2, text: 'Build an app', done: true },
  { id: 3, text: 'Deploy to production', done: false }
];

// Mount each item as an independent component
const components = items.map((item, index) => {
  const target = document.createElement('div');
  target.dataset.index = index;
  container.appendChild(target);

  return mount(TodoItem, {
    target,
    props: item
  });
});
```

```svelte
<!-- TodoItem.svelte -->
<script>
  let { id, text, done = false } = $props();
</script>

<div class:done>
  <input type="checkbox" bind:checked={done} />
  <span>{text}</span>
</div>

<style>
  .done span { text-decoration: line-through; }
</style>
```

**Resulting DOM:**
```html
<div id="todos">
  <div data-index="0"><!-- component 1 --></div>
  <div data-index="1"><!-- component 2 --></div>
  <div data-index="2"><!-- component 3 --></div>
</div>
```

## 9. Mount Inside Existing Element (Tooltip Pattern)

Mount a tooltip/popover into a specific container:

```js
import { mount } from 'svelte';
import Tooltip from './Tooltip.svelte';

// Tooltip container in your HTML
// <div id="tooltip-root"></div>

export function showTooltip(targetElement, content) {
  const tooltipRoot = document.querySelector('#tooltip-root');

  // Mount at specific position
  const tooltip = mount(Tooltip, {
    target: tooltipRoot,
    props: {
      anchor: targetElement,
      content,
      position: 'top'
    }
  });

  return tooltip; // Call unmount(tooltip) to close
}
```

```svelte
<!-- Tooltip.svelte -->
<script>
  import { tick } from 'svelte';

  let { anchor, content, position = 'top' } = $props();
  let tooltipEl;

  $effect(() => {
    // Position after mount
    tick().then(() => {
      if (!tooltipEl || !anchor) return;

      const rect = anchor.getBoundingClientRect();
      const tooltipRect = tooltipEl.getBoundingClientRect();

      if (position === 'top') {
        tooltipEl.style.left = rect.left + rect.width / 2 - tooltipRect.width / 2 + 'px';
        tooltipEl.style.top = rect.top - tooltipRect.height - 8 + 'px';
      }
    });
  });
</script>

<div bind:this={tooltipEl} class="tooltip tooltip-{position}">
  {content}
</div>

<style>
  .tooltip {
    position: fixed;
    z-index: 9999;
    padding: 6px 12px;
    background: #333;
    color: white;
    border-radius: 4px;
    font-size: 14px;
    pointer-events: none;
  }
  .tooltip-top::after {
    content: '';
    position: absolute;
    top: 100%;
    left: 50%;
    transform: translateX(-50%);
    border: 6px solid transparent;
    border-top-color: #333;
  }
</style>
```

## Summary

| API | Use Case |
|-----|----------|
| `mount()` | Initial client render, replacing content |
| `hydrate()` | Activate SSR HTML, preserve structure |
| `render()` | Generate HTML on server |
| `unmount()` | Remove component, optionally after outro |
| `flushSync()` | Force sync updates in tests |
| `tick()` | Wait for async DOM update |
