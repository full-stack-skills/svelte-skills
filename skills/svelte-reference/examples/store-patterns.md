# Store Patterns

## 1. Creating a Writable Store

```js
import { writable } from 'svelte/store';

const count = writable(0);

// Set a new value
count.set(1);

// Update based on current value
count.update(n => n + 1);
```

**In a Svelte component:**

```svelte
<script>
  import { writable } from 'svelte/store';

  const name = writable('World');

  function handleInput(e) {
    name.set(e.target.value);
  }
</script>

<input bind:value={$name} on:input={handleInput} />
<p>Hello {$name}!</p>
```

---

## 2. Creating a Readable Store

```js
import { readable } from 'svelte/store';

// Starts with 'initial', then 'received'
const time = readable('initial', (set) => {
  set('received');

  const interval = setInterval(() => {
    set(new Date().toLocaleTimeString());
  }, 1000);

  // Cleanup function
  return () => clearInterval(interval);
});
```

**Use case: Always-up-to-date values you cannot write to directly**

```js
import { readable } from 'svelte/store';

// Mouse position store
const mousePos = readable({ x: 0, y: 0 }, (set) => {
  function handleMouseMove(e) {
    set({ x: e.clientX, y: e.clientY });
  }

  window.addEventListener('mousemove', handleMouseMove);

  return () => window.removeEventListener('mousemove', handleMouseMove);
});
```

---

## 3. Creating a Derived Store

```js
import { writable, derived } from 'svelte/store';

const firstName = writable('John');
const lastName = writable('Doe');

// Derive a computed value
const fullName = derived(
  [firstName, lastName],
  ([$firstName, $lastName]) => $firstName + ' ' + $lastName
);
```

**Using object shorthand:**

```js
const a = writable(1);
const b = writable(2);
const sum = derived({ a, b }, ({ a, b }) => a + b);
```

**Derived with initial value:**

```js
const price = writable(100);
const tax = writable(0.1);
const total = derived(
  [price, tax],
  ([$price, $tax]) => $price * (1 + $tax),
  110 // initial value before first subscription
);
```

---

## 4. Auto-Subscribe with $ Prefix

```svelte
<script>
  import { writable } from 'svelte/store';

  const count = writable(0);
</script>

<!-- $count auto-subscribes and unsubscribes -->
<button on:click={() => $count++}>
  Count: {$count}
</button>
```

**Works in expressions:**

```svelte
<script>
  const items = writable(['apple', 'banana']);
</script>

<p>{$items.length} items</p>
<ul>
  {#each $items as item}
    <li>{item}</li>
  {/each}
</ul>
```

---

## 5. Store.get() for One-Time Read

```js
import { writable, get } from 'svelte/store';

const count = writable(5);

// Synchronous read without subscribing
console.log(get(count)); // 5
```

**Use case: Reading inside event handlers or callbacks**

```js
import { writable, get } from 'svelte/store';

const user = writable(null);

async function saveUser() {
  const currentUser = get(user); // Get current value
  await api.save(currentUser);
}
```

**Caution:** `get()` creates a temporary subscription - avoid in hot paths.

---

## 6. Store.update() Method

```js
import { writable } from 'svelte/store';

const score = writable(0);

// Functional update
score.update(n => n + 10);

// Update with multiple fields via object
const user = writable({ name: 'Alice', age: 30 });

user.update(u => ({ ...u, age: u.age + 1 }));

// Toggle boolean
const visible = writable(true);
visible.update(v => !v);
```

---

## 7. Store.subscribe Returns Unsubscribe

```js
import { writable } from 'svelte/store';

const count = writable(0);

// Subscribe returns a cleanup function
const unsubscribe = count.subscribe(value => {
  console.log('Count:', value);
});

count.set(1); // Logs: Count: 1
count.set(2); // Logs: Count: 2

// Call to stop listening
unsubscribe();

count.set(3); // Nothing logged (no active subscription)
```

**In component lifecycle:**

```svelte
<script>
  import { onMount, onDestroy } from 'svelte';
  import { writable } from 'svelte/store';

  const data = writable([]);

  let unsubscribe;

  onMount(() => {
    unsubscribe = data.subscribe(value => {
      console.log('Data changed:', value);
    });
  });

  onDestroy(() => {
    if (unsubscribe) unsubscribe();
  });
</script>
```

---

## 8. Cross-Component Store Sharing

**stores.js - Shared store module:**

```js
import { writable } from 'svelte/store';

export const theme = writable('light');
export const user = writable(null);
export const notifications = writable([]);
```

**Component A - Setting value:**

```svelte
<script>
  import { theme } from './stores.js';
</script>

<button on:click={() => theme.set('dark')}>
  Dark Mode
</button>
```

**Component B - Reading value:**

```svelte
<script>
  import { theme } from './stores.js';
</script>

<div class={$theme}>
  Current theme: {$theme}
</div>
```

---

## 9. Store in .svelte.js Module

**lib/stores.svelte.js - Using Svelte 5 runes in stores:**

```js
// stores.svelte.js
import { writable } from 'svelte/store';

function createCountStore() {
  const { subscribe, set, update } = writable(0);

  return {
    subscribe,
    increment: () => update(n => n + 1),
    decrement: () => update(n => n - 1),
    reset: () => set(0)
  };
}

export const count = createCountStore();
```

**Usage in component:**

```svelte
<script>
  import { count } from '$lib/stores.svelte.js';
</script>

<button onclick={count.increment}>+</button>
<span>{$count}</span>
<button onclick={count.decrement}>-</button>
```

---

## Summary

| Pattern | When to Use |
|---------|-------------|
| `writable(initial)` | State that components will modify |
| `readable(initial, start)` | External values (timers, sensors) |
| `derived(stores, fn)` | Computed/transformed values |
| `$storeName` | Auto-subscribe in Svelte components |
| `get(store)` | One-time sync read outside reactive context |
| `store.subscribe()` | Manual subscription control |
| `store.update(fn)` | Modify store value based on previous value |
