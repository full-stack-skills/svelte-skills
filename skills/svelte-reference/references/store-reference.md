# Store API Reference

## Overview

```js
import { writable, readable, derived, readonly, get } from 'svelte/store';
```

Stores are reactive state containers with a consistent contract:

```ts
interface WritableStore<T> {
  subscribe: (run: (value: T) => void) => () => void;
  set: (value: T) => void;
  update: (fn: (value: T) => T) => void;
}

interface ReadableStore<T> {
  subscribe: (run: (value: T) => void) => () => void;
  // No set or update
}
```

---

## writable(initialValue, start?)

Creates a store whose value can be changed via `set` or `update`.

**Signature:**
```ts
function writable<T>(
  initialValue: T,
  start?: (set: (value: T) => void) => () => void
): WritableStore<T>
```

**Parameters:**
- `initialValue` - The starting value
- `start` (optional) - Called when first subscriber subscribes. Return cleanup function.

**Returns:** `{ subscribe, set, update }`

**Example:**
```js
const count = writable(0);
count.set(1);
count.update(n => n + 1);
```

---

## readable(initialValue, start)

Creates a store that cannot be modified externally. Value is set via the `start` callback.

**Signature:**
```ts
function readable<T>(
  initialValue: T,
  start: (set: (value: T) => void) => () => void
): ReadableStore<T>
```

**Parameters:**
- `initialValue` - The starting value (until start() runs)
- `start` - Called with a `set` function. Return cleanup function.

**Example:**
```js
const time = readable('loading', (set) => {
  const interval = setInterval(() => {
    set(new Date().toLocaleTimeString());
  }, 1000);
  return () => clearInterval(interval);
});
```

---

## derived(stores, fn, initial?)

Creates a store whose value is computed from other stores.

**Signature:**
```ts
function derived<
  S extends Store<any>,
  T
>(
  stores: S | [S] | [S, S] | [S, S, S] | ...,
  fn: (values: any[], set: (value: T) => void) => () => void | T,
  initialValue?: T
): ReadableStore<T>
```

**Single store:**
```js
const doubled = derived(count, $count => $count * 2);
```

**Multiple stores (array):**
```js
const sum = derived(
  [a, b],
  ([$a, $b]) => $a + $b
);
```

**Object shorthand:**
```js
// Equivalent to: derived([a, b], ([$a, $b]) => ...)
const sum = derived({ a, b }, ({ a, b }) => a + b);
```

**With start/cleanup function:**
```js
const filtered = derived(items, ($items, set) => {
  const filtered = $items.filter(x => x.active);
  set(filtered);

  // Optional: return cleanup
  return () => { /* cleanup */ };
});
```

---

## get(store)

Synchronously read a store's value without subscribing.

**Signature:**
```ts
function get<T>(store: ReadableStore<T>): T
```

**Example:**
```js
const count = writable(42);
console.log(get(count)); // 42
```

**Caution:** Creates a temporary subscription. Avoid in hot paths or frequently called code.

---

## store.subscribe() Return Value

`subscribe` returns an unsubscribe function that must be called to prevent memory leaks.

**Signature:**
```ts
subscribe: (run: (value: T) => void) => () => void
```

**Example:**
```js
const count = writable(0);

const unsubscribe = count.subscribe(value => {
  console.log(value);
});

count.set(1); // logs: 1
count.set(2); // logs: 2

unsubscribe(); // stop listening
count.set(3); // nothing logged
```

---

## Store update(fn) Pattern

`update` transforms the current value using a function.

**Signature:**
```ts
update: (fn: (value: T) => T) => void
```

**Examples:**
```js
const count = writable(0);

// Increment
count.update(n => n + 1);

// Reset to zero
count.update(() => 0);

// Complex update
count.update(n => ({
  ...n,
  count: n.count + 1
}));
```

---

## set vs update

| Method | Use Case |
|--------|----------|
| `set(value)` | Replace with a specific value |
| `update(fn)` | Transform based on current value |

```js
const user = writable({ name: 'Alice', age: 30 });

// set - replace entirely
user.set({ name: 'Bob', age: 25 });

// update - modify based on current
user.update(u => ({ ...u, age: u.age + 1 }));
```

---

## readonly(store)

Returns a read-only version of a writable store.

**Signature:**
```ts
function readonly<T>(store: WritableStore<T>): ReadableStore<T>
```

**Example:**
```js
const writableCount = writable(0);
const readonlyCount = readonly(writableCount);

readonlyCount.set(1); // Error! set does not exist
```

---

## Store Contract

All stores (including custom) must implement this interface:

```ts
interface Store<T> {
  subscribe: (run: (value: T) => void) => () => void;
}
```

Optional for writable stores:
```ts
interface WritableStore<T> extends Store<T> {
  set: (value: T) => void;
  update: (fn: (value: T) => T) => void;
}
```
