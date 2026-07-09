# Mount, Unmount, and Hydrate Reference

Complete technical reference for Svelte 5's imperative component APIs.

## mount() API

### Signature

```ts
function mount<T extends Component>(
  component: T,
  options: {
    target: Element;
    props?: Record<string, unknown>;
    events?: Record<string, (event: CustomEvent) => void>;
  }
): ReturnType<T>;
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `component` | `Component` | Yes | Svelte component to mount |
| `target` | `Element` | Yes | DOM element to mount into |
| `props` | `Record<string, unknown>` | No | Initial props passed to component |
| `events` | `Record<string, Function>` | No | Event handler proxies |

### Return Value

Returns the component instance with:
- All exported methods/properties
- Event bindings via `events` option
- Internal `$state` and `$derived` accessible via instance

### Basic Usage

```js
import { mount } from 'svelte';
import App from './App.svelte';

const instance = mount(App, {
  target: document.querySelector('#app'),
  props: {
    user: { name: 'Ada' },
    onNavigate: (e) => console.log('navigate', e.detail)
  },
  events: {
    // Map Svelte component events to callbacks
    ready: (e) => console.log('ready'),
    update: (e) => handleUpdate(e.detail)
  }
});
```

### Target Element Behavior

- **Replaces content**: Any existing children of `target` are replaced
- **Multiple mounts**: Each `mount()` call creates independent instances
- **No cleanup of target**: Target element itself remains (just its contents change)

```js
// These are two independent mounted instances
const app1 = mount(ComponentA, { target: document.querySelector('#region-1') });
const app2 = mount(ComponentA, { target: document.querySelector('#region-2') });
```

## What Happens During Mount

### Mount Phase Order

1. **Component constructor** - Create reactive state (`$state`, `$derived`)
2. **Props assignment** - Apply `props` object to component props
3. **DOM creation** - Generate initial HTML from component template
4. **Insert into DOM** - Append to target element
5. **Return instance** - Immediately return to caller

### Effects Are NOT Run During Mount

**Critical**: Effects scheduled via `$effect` do NOT execute during `mount()`.

```js
import { mount } from 'svelte';
import MyComponent from './MyComponent.svelte';

const app = mount(MyComponent, { target: document.body });

// At this point:
// - DOM is rendered
// - $effect callbacks have NOT yet run
// - They will run when the microtask queue processes
```

**Why?** Effects are triggered by a microtask, allowing:
- Batching multiple state changes
- Avoiding redundant renders during initialization
- Consistent behavior with reactive updates

**Testing implication:**

```js
test('effect runs after mount', async () => {
  const app = mount(EffectsComponent, { target: document.body });

  // DOM exists but effect hasn't fired yet
  expect(effectRan).toBe(false);

  // Wait for microtask
  await Promise.resolve();

  // Now effect has run
  expect(effectRan).toBe(true);
});
```

## unmount() API

### Signature

```ts
function unmount(
  app: ComponentInstance,
  options?: {
    outro?: boolean;
  }
): void;
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `app` | `ComponentInstance` | Yes | Instance returned from `mount()` or `hydrate()` |
| `options.outro` | `boolean` | No | Wait for CSS transitions (default: `false`) |

### Without Outro (Default)

```js
import { mount, unmount } from 'svelte';

const app = mount(Modal, { target: document.body });
// Immediately removes from DOM
unmount(app);
```

The component is removed synchronously - no animations play.

### With Outro

```js
unmount(app, { outro: true });
```

**Behavior:**
1. Component's `$effect` cleanup functions run immediately
2. CSS transitions/animations play
3. Svelte waits for `animationend` / `transitionend` events
4. After all transitions complete, DOM element is removed
5. Default timeout: 1000ms (prevents infinite animation hangs)

**Cleanup function timing:**

```svelte
<script>
  import { onMount } from 'svelte';

  onMount(() => {
    // This runs BEFORE outro animations
    return () => console.log('cleanup');
  });
</script>

<div transition:fade>
  Content
</div>
```

### Multiple Instances

Each mounted instance is independent:

```js
const instances = components.map(c => mount(c, { target: createTarget() }));

// Unmount only the second one
unmount(instances[1]);

// First and third still mounted
```

## hydrate() API

### Signature

```ts
function hydrate<T extends Component>(
  component: T,
  options: {
    target: Element;
    props?: Record<string, unknown>;
    events?: Record<string, (event: CustomEvent) => void>;
  }
): ReturnType<T>;
```

Same signature as `mount()`.

### hydrate() vs mount()

| Aspect | `mount()` | `hydrate()` |
|--------|-----------|-------------|
| Target content | **Replaced** | **Preserved + activated** |
| Use case | Fresh client render | Activate SSR HTML |
| SSR HTML required | No | Yes (identical structure) |
| Event binding | Full re-attachment | Attach to existing elements |
| Performance | Full render | Reuses existing DOM |

### When to Use hydrate()

```js
// SvelteKit client-side navigation
import { hydrate } from 'svelte';
import Page from './Page.svelte';

// HTML already in DOM from SSR
const app = hydrate(Page, {
  target: document.querySelector('#app'),
  props: { data: window.__DATA__ }
});
```

### Hydration Process

1. Find target element with existing SSR HTML
2. Parse component template
3. Compare existing DOM structure
4. Attach event listeners to matching elements
5. Initialize reactive state from props
6. Return interactive instance

**Mismatches cause errors:**

```js
// Server renders: <div class="card">Content</div>
// Client expects:  <div class="panel">Content</div>
// ^^^^^^^^^^^^^^^^^^ HYDRATION MISMATCH ERROR
```

## Multiple Component Instances

### Independent State

Each `mount()` creates completely independent state:

```js
const app1 = mount(Counter, { target: div1, props: { initial: 0 } });
const app2 = mount(Counter, { target: div2, props: { initial: 100 } });

// app1.count = 0, app2.count = 100
// Changing one doesn't affect the other
```

### Shared State via Context

For shared state across components:

```js
import { mount } from 'svelte';
import { setContext } from 'svelte';
import AppShell from './AppShell.svelte';
import SharedState from './state.svelte.js';

// Create shared state
const state = SharedState();
setContext('app-state', state);

// Mount app - children can getContext('app-state')
const app = mount(AppShell, { target: document.body });
```

### Event Handling Between Instances

```js
import { mount } from 'svelte';

const instances = items.map((item, i) => {
  const target = document.createElement('div');
  document.querySelector('#list').appendChild(target);

  return mount(ListItem, {
    target,
    props: {
      item,
      onSelect: () => selectItem(i)
    }
  });
});

function selectItem(index) {
  // Update all instances
  instances.forEach((inst, i) => {
    inst.setActive(i === index);
  });
}
```

## flushSync() Reference

### Signature

```ts
function flushSync(): void;
```

### Purpose

Forces the Svelte runtime to process all pending reactive updates **synchronously** immediately.

### When to Use

- **Testing**: Verify state changes reflect in DOM immediately
- **Synchronization**: External libraries that need current DOM state
- **Debugging**: Step through updates synchronously

### Without flushSync

```js
let value = $state(0);

$effect(() => {
  console.log(value); // Logs 0, not 1 yet!
});

value = 1;
// Microtask hasn't run, effect hasn't triggered
```

### With flushSync

```js
let value = $state(0);

$effect(() => {
  console.log(value); // Will log 1
});

value = 1;
flushSync(); // Process all pending updates now
console.log(value); // 1
```

## tick() Reference

### Signature

```ts
function tick(): Promise<void>;
```

### Purpose

Returns a promise that resolves after Svelte has applied all pending DOM updates.

### Use Cases

1. **After programmatic state changes** - Wait for DOM to reflect changes
2. **Focus management** - Focus after element is in DOM
3. **Measurements** - Get accurate element dimensions after changes

### Example

```js
import { tick } from 'svelte';

async function updateAndMeasure() {
  list = [...list, newItem]; // Trigger reactive update

  await tick(); // Wait for DOM update

  // Now scrollHeight reflects new content
  const height = container.scrollHeight;
}
```

### tick() vs Promise.resolve()

| Aspect | `tick()` | `Promise.resolve()` |
|--------|----------|---------------------|
| Waits for Svelte updates | Yes | No |
| DOM accurate | Yes | No |
| Use case | Svelte reactivity | Generic async |

## Summary Table

| API | Description | Key Behavior |
|-----|-------------|--------------|
| `mount()` | Mount component | Replaces target content, async effects |
| `unmount()` | Remove component | Optional outro wait |
| `hydrate()` | Activate SSR | Preserves existing DOM |
| `flushSync()` | Force sync update | Processes all pending updates |
| `tick()` | Wait for DOM update | Resolves after DOM changes |
