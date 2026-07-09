# Store Reference - Complete Guide

Complete reference for Svelte stores in both Svelte 4 and Svelte 5.

---

## Overview

Stores provide a way to manage shared state across components. They work identically in both Svelte 4 and Svelte 5.

---

## Store Types

### Writable Store

Most common store type - can be written to from anywhere.

```javascript
import { writable } from 'svelte/store';

export const count = writable(0);
export const user = writable({ name: 'Alice', age: 30 });
export const items = writable([]);
```

### Readable Store

Cannot be written to from outside - created with an initial value and optional start/stop functions.

```javascript
import { readable } from 'svelte/store';

// Simple readable with initial value
const time = readable(new Date());

// Readable with auto-updating start function
const elapsed = readable(0, (set, update) => {
  const start = Date.now();
  const interval = setInterval(() => {
    set(Date.now() - start);
  }, 1000);

  // Return cleanup function
  return () => clearInterval(interval);
});
```

### Derived Store

Creates a store whose value is computed from other stores.

```javascript
import { writable, derived } from 'svelte/store';

const a = writable(1);
const b = writable(2);

// Simple derived
const sum = derived([a, b], ([$a, $b]) => $a + $b);

// With initial value
const total = writable(0);
const count = writable(10);
const average = derived([total, count], ([$total, $count]) =>
  $count === 0 ? 0 : $total / $count
);
```

---

## Store Contract

A valid store must implement the **store contract**:

```javascript
{
  subscribe: (callback) => () => void
}
```

### Minimal Store Example

```javascript
function createCounter() {
  let value = 0;

  const subscribers = new Set();

  function subscribe(callback) {
    subscribers.add(callback);
    callback(value);  // Immediately call with current value
    return () => subscribers.delete(callback);  // Return unsubscribe function
  }

  function set(newValue) {
    value = newValue;
    subscribers.forEach(cb => cb(value));
  }

  function update(fn) {
    set(fn(value));
  }

  return { subscribe, set, update };
}

export const counter = createCounter();
```

### Auto-Subscribe Behavior

The `$` prefix in templates automatically:
1. Subscribes to the store
2. Returns the current value
3. Unsubscribes when component unmounts

```svelte
<script>
  import { counter } from './counter.js';
</script>

<!-- $counter auto-subscribes -->
<p>{$counter}</p>
```

---

## get() Function

Read current store value without subscribing.

```javascript
import { writable, get } from 'svelte/store';

const count = writable(0);

// Get current value (non-reactive)
const currentValue = get(count);
```

**Note:** Avoid using `get()` in templates - use `$store` instead for reactivity.

---

## Custom Store Implementation

### Store with Actions

```javascript
function createStore(initialValue) {
  const { subscribe, set, update } = writable(initialValue);

  return {
    subscribe,

    // Actions
    increment: () => update(n => n + 1),
    decrement: () => update(n => n - 1),
    reset: () => set(initialValue),

    // Generic setter
    setValue: (newValue) => set(newValue),

    // Generic updater
    updateValue: (fn) => update(fn),

    // Async action
    async fetchData() {
      update(state => ({ ...state, loading: true }));
      try {
        const data = await fetch('/api/data').then(r => r.json());
        set({ data, loading: false });
      } catch (error) {
        set({ data: null, loading: false, error });
      }
    }
  };
}

export const counter = createStore(0);
export const dataStore = createStore({ data: null, loading: false, error: null });
```

### Store with Persistence

```javascript
function persistentStore(key, initialValue) {
  // Load from localStorage if available
  const storedValue = typeof localStorage !== 'undefined'
    ? localStorage.getItem(key)
    : null;

  const initial = storedValue ? JSON.parse(storedValue) : initialValue;
  const store = writable(initial);

  // Save to localStorage on every change
  store.subscribe(value => {
    if (typeof localStorage !== 'undefined') {
      localStorage.setItem(key, JSON.stringify(value));
    }
  });

  return store;
}

export const theme = persistentStore('theme', 'light');
export const preferences = persistentStore('preferences', {
  language: 'en',
  notifications: true
});
```

---

## Store in .svelte.js

In Svelte 5, `.svelte.js` files can contain runes or stores:

```javascript
// stores.svelte.js
import { writable, derived } from 'svelte/store';

export const appStore = writable({
  initialized: false,
  user: null
});

export const isLoggedIn = derived(
  appStore,
  $app => $app.user !== null
);
```

```javascript
// state.svelte.js - Mix runes and stores
import { writable } from 'svelte/store';

// Store for cross-component state
export const sharedState = writable('shared');

// Runes for local component state
export function createLocalState(initial) {
  return { value: $state(initial) };
}
```

---

## Store and SSR

### Client-Side Only Store

```javascript
import { writable } from 'svelte/store';

function browserStore(initialValue) {
  const isBrowser = typeof window !== 'undefined';

  if (!isBrowser) {
    // Return a no-op store for SSR
    return {
      subscribe: () => () => {},
      set: () => {},
      update: () => {}
    };
  }

  return writable(initialValue);
}

export const localStorageStore = browserStore(null);
```

### Hydration-Safe Store Usage

```svelte
<script>
  import { store } from './stores/store.js';
  import { onMount } from 'svelte';

  let mounted = false;
  onMount(() => {
    mounted = true;
  });
</script>

{#if mounted}
  <p>Value: {$store}</p>
{:else}
  <p>Loading...</p>
{/if}
```

### SSR Store Initialization

```javascript
// stores/ssrSafe.js
import { writable } from 'svelte/store';

export function createSSRStore(initialValue) {
  // On server: initialize but don't hydrate
  const store = writable(initialValue);

  // On client: restore from server-rendered state
  if (typeof window !== 'undefined') {
    // Called by app initialization
    store.hydrate = (serverValue) => {
      store.set(serverValue);
    };
  }

  return store;
}
```

---

## Store Update Patterns

### Functional Update

```javascript
store.update(state => ({
  ...state,
  count: state.count + 1
}));
```

### Batch Update

```javascript
store.update(state => {
  const newState = { ...state };
  for (let i = 0; i < 1000; i++) {
    newState.items.push({ id: i, value: i });
  }
  return newState;
});
```

### Async Update

```javascript
export const data = writable({ loading: false, items: [], error: null });

export async function loadData() {
  data.update(s => ({ ...s, loading: true }));

  try {
    const response = await fetch('/api/data');
    const items = await response.json();
    data.set({ loading: false, items, error: null });
  } catch (error) {
    data.update(s => ({ ...s, loading: false, error: error.message }));
  }
}
```

---

## Store Subscriptions

### Manual Subscribe

```javascript
import { count } from './stores/count.js';

const unsubscribe = count.subscribe(value => {
  console.log('count changed:', value);
});

// Later: unsubscribe
unsubscribe();
```

### Store in Reactive Context

```svelte
<script>
  import { count } from './stores/count.js';

  // Svelte 4: $: double = $count * 2;
  // Svelte 5:
  let double = $derived($count * 2);
</script>

<p>{$count} x 2 = {double}</p>
```

### Multiple Store Subscription

```svelte
<script>
  import { user } from './stores/user.js';
  import { settings } from './stores/settings.js';
  import { todos } from './stores/todos.js';
</script>

<h1>Welcome, {$user.name}</h1>
<p>Theme: {$settings.theme}</p>
<p>Todos: {$todos.length}</p>
```

---

## Built-in Store Utilities

### reset()

Not a built-in - but commonly implemented pattern:

```javascript
function createStore(initialValue) {
  const { subscribe, set } = writable(initialValue);

  return {
    subscribe,
    reset: () => set(initialValue),
    // ... other methods
  };
}
```

### stores (deprecated)

```javascript
import { stores } from 'svelte/store';
// Deprecated in Svelte 5 - not recommended for new code
```

---

## Common Patterns

### Store for Global State

```javascript
// stores/app.js
import { writable, derived } from 'svelte/store';

export const user = writable(null);
export const isAuthenticated = derived(user, $user => $user !== null);
export const permissions = writable([]);

export function logout() {
  user.set(null);
  permissions.set([]);
}
```

### Store for Component State

```javascript
// stores/listStore.js
import { writable, derived } from 'svelte/store';

export function createListStore() {
  const items = writable([]);
  const filter = writable('');

  const filtered = derived([items, filter], ([$items, $filter]) =>
    $filter
      ? $items.filter(item => item.name.includes($filter))
      : $items
  );

  return {
    items,
    filter,
    filtered,

    add: (item) => items.update(i => [...i, item]),
    remove: (id) => items.update(i => i.filter(item => item.id !== id)),
    setFilter: (f) => filter.set(f),
    clear: () => items.set([])
  };
}
```

### Store for API State

```javascript
// stores/apiStore.js
import { writable, derived } from 'svelte/store';

function createApiStore(fetcher) {
  const data = writable(null);
  const loading = writable(false);
  const error = writable(null);

  const state = derived(
    [data, loading, error],
    ([$data, $loading, $error]) => ({
      data: $data,
      loading: $loading,
      error: $error,
      ready: $data !== null && !$loading
    })
  );

  async function fetch(...args) {
    loading.set(true);
    error.set(null);
    try {
      const result = await fetcher(...args);
      data.set(result);
      return result;
    } catch (e) {
      error.set(e);
      throw e;
    } finally {
      loading.set(false);
    }
  }

  return {
    data,
    loading,
    error,
    state,
    fetch
  };
}

// Usage
export const usersStore = createApiStore(() =>
  fetch('/api/users').then(r => r.json())
);
```

---

## Performance Considerations

### Avoid Unnecessary Updates

```javascript
// ❌ Bad: Creates new array on every update
export const items = writable([]);
function addItem(item) {
  items.update(i => [...i, item]);  // Always triggers subscribers
}

// ✅ Better: Only update when value actually changes
export const items = writable([]);

function addItem(item) {
  items.update(i => {
    const newItems = [...i, item];
    return newItems;  // Still triggers, but cleaner
  });
}
```

### Use Derived for Computed Values

```javascript
// ❌ Bad: Computed in component
$: filteredItems = items.filter(i => i.active);

// ✅ Better: Derived store
const filteredItems = derived(items, $items =>
  $items.filter(i => i.active)
);
```

### Unsubscribe When Done

```javascript
import { onDestroy } from 'svelte';

onDestroy(unsubscribe);
```
