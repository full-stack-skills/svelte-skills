# Store Comparison - Svelte 4 and Svelte 5

Stores work identically in both Svelte 4 and Svelte 5. This guide shows patterns for both modes.

## Svelte 4 Store Patterns

### Creating a Writable Store

```javascript
// stores/counter.js
import { writable } from 'svelte/store';

export const counter = writable(0);

// Helper functions
export function increment() {
  counter.update(n => n + 1);
}

export function reset() {
  counter.set(0);
}
```

### Using a Store in Svelte 4

```svelte
<!-- Counter.svelte -->
<script>
  import { counter, increment, reset } from './stores/counter.js';
</script>

<p>Count: {$counter}</p>
<button on:click={increment}>+1</button>
<button on:click={reset}>Reset</button>
```

### Derived Store

```javascript
// stores/todos.js
import { writable, derived } from 'svelte/store';

export const todos = writable([
  { text: 'Learn Svelte', done: true },
  { text: 'Build an app', done: false }
]);

export const doneCount = derived(todos, $todos =>
  $todos.filter(t => t.done).length
);

export const remainingCount = derived(todos, $todos =>
  $todos.filter(t => !t.done).length
);
```

### Using Derived Stores

```svelte
<script>
  import { todos, doneCount, remainingCount } from './stores/todos.js';
</script>

<p>Done: {doneCount} / Remaining: {remainingCount}</p>

<ul>
  {#each $todos as todo}
    <li class:done={todo.done}>{todo.text}</li>
  {/each}
</ul>
```

### Readable Store

```javascript
// stores/time.js
import { readable } from 'svelte/store';

export const time = readable(new Date(), (set) => {
  const interval = setInterval(() => {
    set(new Date());
  }, 1000);

  return () => clearInterval(interval);
});
```

```svelte
<script>
  import { time } from './stores/time.js';
</script>

<p>{$time.toLocaleTimeString()}</p>
```

### Custom Store with Subscribe Method

```javascript
// stores/createStore.js
export function createStore(initialValue) {
  let value = initialValue;
  const { subscribe, set, update } = writable(value);

  return {
    subscribe,
    increment: () => update(n => n + 1),
    decrement: () => update(n => n - 1),
    reset: () => set(initialValue),
    setValue: (newValue) => set(newValue)
  };
}
```

```svelte
<script>
  import { createStore } from './stores/createStore.js';

  const counter = createStore(10);
</script>

<p>Count: {$counter}</p>
<button on:click={counter.increment}>+1</button>
<button on:click={counter.decrement}>-1</button>
<button on:click={counter.reset}>Reset</button>
```

---

## Svelte 5 Store Patterns

### Stores Still Work in Svelte 5!

```svelte
<!-- Counter.svelte - Svelte 5 -->
<script>
  import { counter, increment, reset } from './stores/counter.js';
  // $counter syntax is identical
</script>

<p>Count: {$counter}</p>
<button onclick={increment}>+1</button>
<button onclick={reset}>Reset</button>
```

> **Key Point:** The `$store` auto-subscribe syntax and store behavior are identical in Svelte 5. No migration needed for stores!

### Using Stores with Runes Components

```svelte
<!-- RunesComponent.svelte -->
<script>
  import { counter } from './stores/counter.js';
  import { someRunesState } from './state.js';

  // Mix runes state and stores
  let localValue = $state(0);
</script>

<p>Store: {$counter}</p>
<p>Local: {localValue}</p>
<p>Runes: {someRunesState}</p>
```

### Store Updates from Callbacks

```svelte
<script>
  import { counter } from './stores/counter.js';

  function handleUpdate(newValue) {
    // Direct store methods still work
    counter.set(newValue);
  }

  function handleIncrement() {
    // Or use update
    counter.update(n => n + 1);
  }
</script>
```

---

## $store Auto-Subscribe Syntax

### Basic Usage (Both Modes)

```svelte
<script>
  import { myStore } from './store.js';
</script>

<!-- $ prefix auto-subscribes -->
<p>{$myStore}</p>
```

### Store in Reactive Context

```svelte
<script>
  import { count } from './stores/count.js';

  // $: double = $count * 2;  // Svelte 4 Legacy
  // Svelte 5: use $derived
  let double = $derived($count * 2);
</script>

<p>Double: {double}</p>
```

### Store as Component Prop

```svelte
<!-- Display.svelte -->
<script>
  export let count;  // Store passed in
</script>

<p>Count: {$count}</p>
```

```svelte
<!-- Parent.svelte -->
<script>
  import { count } from './stores/count.js';
  import Display from './Display.svelte';
</script>

<!-- Pass store reference, not subscribed value -->
<Display {count} />
```

### Subscribing to Multiple Stores

```svelte
<script>
  import { user } from './stores/user.js';
  import { preferences } from './stores/preferences.js';
  import { settings } from './stores/settings.js';
</script>

<p>Welcome, {$user.name}</p>
<p>Theme: {$preferences.theme}</p>
<p>Version: {$settings.version}</p>
```

---

## Custom Store Implementation

### Store with Persistence

```javascript
// stores/persistentStore.js
import { writable } from 'svelte/store';

export function persistentStore(key, initialValue) {
  const storedValue = typeof localStorage !== 'undefined'
    ? localStorage.getItem(key)
    : null;

  const initial = storedValue ? JSON.parse(storedValue) : initialValue;
  const store = writable(initial);

  store.subscribe(value => {
    if (typeof localStorage !== 'undefined') {
      localStorage.setItem(key, JSON.stringify(value));
    }
  });

  return store;
}
```

```javascript
// Usage
import { persistentStore } from './stores/persistentStore.js';

export const theme = persistentStore('theme', 'light');
export const userPreferences = persistentStore('preferences', {
  language: 'en',
  notifications: true
});
```

### Store with Actions

```javascript
// stores/todoStore.js
import { writable, derived } from 'svelte/store';

function createTodoStore() {
  const { subscribe, update } = writable([]);

  return {
    subscribe,

    add: (text) => update(todos => [
      ...todos,
      {
        id: Date.now(),
        text,
        done: false,
        createdAt: new Date()
      }
    ]),

    toggle: (id) => update(todos =>
      todos.map(t => t.id === id ? { ...t, done: !t.done } : t)
    ),

    remove: (id) => update(todos =>
      todos.filter(t => t.id !== id)
    ),

    clearDone: () => update(todos =>
      todos.filter(t => !t.done)
    )
  };
}

export const todos = createTodoStore();
export const doneCount = derived(todos, $todos =>
  $todos.filter(t => t.done).length
);
```

```svelte
<script>
  import { todos, doneCount } from './stores/todoStore.js';

  let newTodo = '';

  function addTodo() {
    if (newTodo.trim()) {
      todos.add(newTodo);
      newTodo = '';
    }
  }
</script>

<form on:submit|preventDefault={addTodo}>
  <input bind:value={newTodo} placeholder="New todo" />
  <button type="submit">Add</button>
</form>

<ul>
  {#each $todos as todo}
    <li>
      <input
        type="checkbox"
        checked={todo.done}
        on:change={() => todos.toggle(todo.id)}
      />
      <span class:done={todo.done}>{todo.text}</span>
      <button on:click={() => todos.remove(todo.id)}>x</button>
    </li>
  {/each}
</ul>

{#if $doneCount > 0}
  <button on:click={todos.clearDone}>Clear done ({$doneCount})</button>
{/if}

<style>
  .done { text-decoration: line-through; }
</style>
```

---

## Store Update Patterns

### Update with Functional Style

```javascript
// Functional updates preserve immutability
store.update(state => ({
  ...state,
  count: state.count + 1
}));
```

### Batch Updates

```javascript
// For performance, batch multiple updates
store.update(state => {
  const newState = { ...state };
  for (let i = 0; i < 1000; i++) {
    newState.items.push({ id: i, value: i });
  }
  return newState;
});
```

### Async Store Updates

```javascript
import { writable } from 'svelte/store';

export const data = writable([]);

export async function fetchData() {
  data.set({ loading: true });

  try {
    const response = await fetch('/api/data');
    const json = await response.json();
    data.set({ loading: false, items: json });
  } catch (error) {
    data.set({ loading: false, error: error.message });
  }
}
```

---

## Store in .svelte.js Files

### Svelte 5: Stores in Module Context

```javascript
// stores/appStore.js
import { writable } from 'svelte/store';

export const appState = writable({
  theme: 'light',
  sidebarOpen: false
});

export const currentUser = writable(null);
```

```javascript
// state.svelte.js - Can use runes but stores still work
import { appState } from './stores/appStore.js';

export const localState = writable('value');
```

### When to Use Stores vs Runes

| Scenario | Use |
|----------|-----|
| Cross-component state | Store |
| Complex derived state | Store (derived) |
| Local component state | Runes ($state) |
| Simple local state | Runes ($state) |
| Shared immutable data | Store (readable) |

---

## SSR Considerations

### Client-Side Only Store

```javascript
// stores/browserStore.js
import { writable } from 'svelte/store';

function createBrowserStore(initialValue) {
  const isBrowser = typeof window !== 'undefined';

  if (!isBrowser) {
    return { subscribe: () => () => {}, set: () => {}, update: () => {} };
  }

  return writable(initialValue);
}

export const localStorageStore = createBrowserStore(null);
```

### Hydration-Safe Stores

```svelte
<script>
  import { store } from './stores/store.js';
  import { onMount } from 'svelte';

  let hydrated = false;

  onMount(() => {
    hydrated = true;
  });
</script>

{#if hydrated}
  <p>Store value: {$store}</p>
{:else}
  <p>Loading...</p>
{/if}
```
