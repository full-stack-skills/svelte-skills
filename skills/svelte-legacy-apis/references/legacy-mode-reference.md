# Legacy Mode Reference - Svelte 5

Complete reference for using Legacy Mode in Svelte 5.

---

## Overview

Legacy Mode allows Svelte 5 to run Svelte 4-style code without runes. This is useful for:
- Gradual migration of large codebases
- Maintaining backward compatibility
- Learning Svelte 4 patterns

---

## Enabling Legacy Mode

### Component-Level: `<svelte:options runes={false}>`

```svelte
<!-- Force legacy mode for this component -->
<svelte:options runes={false} />

<script>
  let count = 0;        // Automatically reactive
  $: doubled = count * 2;
</script>

<button on:click={() => count++}>
  {count} (doubled: {doubled})
</button>
```

### Project-Level: svelte.config.js

```js
// svelte.config.js
export default {
  compilerOptions: {
    runes: false
  }
};
```

### File-Level: .svelte.js files

```javascript
// This file uses runes by default in Svelte 5
// Use legacy mode by NOT using runes
export const count = writable(0);  // writable is a store, not runes
```

---

## What Works in Legacy Mode

### Reactive let (Top-Level)

```svelte
<svelte:options runes={false} />
<script>
  let name = 'Alice';       // Automatically reactive
  let count = 0;            // Same
  let obj = { x: 1 };      // Same - but mutation issues (see below)
</script>
```

### Reactive Statements ($:)

```svelte
<svelte:options runes={false} />
<script>
  let a = 1;
  let b = 2;
  $: sum = a + b;
  $: console.log('sum:', sum);
</script>
```

### export let Props

```svelte
<svelte:options runes={false} />
<script>
  export let name;
  export let age = 18;
</script>
```

### Event Handlers (on:event)

```svelte
<svelte:options runes={false} />
<button on:click={handleClick}>Click</button>
<form on:submit|preventDefault={handleSubmit}>
```

### Slots

```svelte
<svelte:options runes={false} />
<!-- Component.svelte -->
<slot />
<slot name="header" />
```

### createEventDispatcher

```svelte
<svelte:options runes={false} />
<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();
  dispatch('change', { value: 42 });
</script>
```

### Stores ($store syntax)

```svelte
<svelte:options runes={false} />
<script>
  import { count } from './stores.js';
</script>
<p>{$count}</p>
```

---

## What Doesn't Work in Legacy Mode

### $state (Runes)

```svelte
<svelte:options runes={false} />
<script>
  let count = $state(0);  // ❌ Compile error!
</script>
```

### $derived

```svelte
<svelte:options runes={false} />
<script>
  let doubled = $derived(count * 2);  // ❌ Compile error!
</script>
```

### $effect

```svelte
<svelte:options runes={false} />
<script>
  $effect(() => console.log('mounted'));  // ❌ Compile error!
</script>
```

### $props

```svelte
<svelte:options runes={false} />
<script>
  let { name } = $props();  // ❌ Compile error!
</script>
```

### {@render}

```svelte
<svelte:options runes={false} />
<script>
  let { children } = $props();
</script>
{@render children()}  // ❌ Compile error!
```

---

## Critical Differences: Array/Object Mutation

### Legacy Mode (Assignment-Based Reactivity)

In Legacy mode, Svelte tracks assignments, NOT mutations:

```svelte
<svelte:options runes={false} />
<script>
  let obj = { count: 0 };
  let arr = [1, 2, 3];

  function incrementObj() {
    obj.count++;  // ❌ Does NOT trigger update!
    obj = obj;    // ✅ Required: reassign
  }

  function addToArr() {
    arr.push(4);  // ❌ Does NOT trigger update!
    arr = arr;    // ✅ Required: reassign
  }
</script>
```

### Runes Mode (Mutation Works)

```svelte
<script>
  let obj = $state({ count: 0 });
  let arr = $state([1, 2, 3]);

  function incrementObj() {
    obj.count++;  // ✅ Works!
  }

  function addToArr() {
    arr.push(4);  // ✅ Works!
  }
</script>
```

### Workarounds in Legacy Mode

```svelte
<svelte:options runes={false} />
<script>
  let items = [1, 2, 3];

  // Array methods need reassignment
  function add(item) {
    items = [...items, item];  // ✅ Create new array
  }

  function remove(index) {
    items = items.filter((_, i) => i !== index);  // ✅
  }

  function update(index, value) {
    items = items.map((item, i) => i === index ? value : item);  // ✅
  }

  // Object mutations need reassignment
  let user = { name: 'Alice', age: 30 };

  function birthday() {
    user = { ...user, age: user.age + 1 };  // ✅
  }
</script>
```

---

## Mixing Legacy and Runes Components

### Interoperability

Svelte 5 allows seamless mixing of legacy and runes components:

```svelte
<!-- LegacyComponent.svelte -->
<svelte:options runes={false} />
<script>
  export let title;
  let count = 0;
</script>
<p>{title}: {count}</p>
<button on:click={() => count++}>+</button>
```

```svelte
<!-- RunesComponent.svelte -->
<script>
  let { title, children } = $props();
  let count = $state(0);
</script>
<p>{title}: {count}</p>
<button onclick={() => count++}>+</button>
{@render children?.()}
```

```svelte
<!-- App.svelte (uses runes) -->
<script>
  import LegacyComponent from './LegacyComponent.svelte';
  import RunesComponent from './RunesComponent.svelte';
</script>

<!-- Both work together! -->
<LegacyComponent title="Legacy" />
<RunesComponent title="Runes">
  <p>Content from runes parent</p>
</RunesComponent>
```

### Communication Patterns

**Legacy child with Runes parent:**
```svelte
<!-- RunesParent.svelte -->
<script>
  import LegacyChild from './LegacyChild.svelte';
  let initial = $state('Hello');
</script>

<LegacyChild text={initial} />
```

**Runes child with Legacy parent:**
```svelte
<!-- LegacyParent.svelte -->
<svelte:options runes={false} />
<script>
  import RunesChild from './RunesChild.svelte';
  let result = '';
</script>

<RunesChild onResult={(val) => result = val} />
<p>Result: {result}</p>
```

---

## When to Use Legacy Mode

### Use Legacy Mode When:

1. **Large Svelte 4 codebase** - Gradual migration is more practical
2. **Team unfamiliar with runes** - Allows incremental learning
3. **Third-party components** - Svelte 4 libraries you can't modify
4. **Temporary compatibility layer** - Waiting for library updates

### Don't Use Legacy Mode When:

1. **New project** - Start with runes for better performance
2. **Performance-critical code** - Runes are more optimized
3. **Need new features** - Some Svelte 5 features only work with runes
4. **Full control over codebase** - Migrate fully for long-term benefits

---

## Migration Path

### Step 1: Isolate

```svelte
<!-- Mark legacy components explicitly -->
<svelte:options runes={false} />
```

### Step 2: Document

```svelte
<!-- Add migration notes -->
<svelte:options runes={false} />
<!--
  TODO: Migrate to runes
  - $: → $derived
  - export let → $props
  - on: → onclick
-->
```

### Step 3: Migrate Incrementally

```svelte
<!-- Remove runes={false} after migrating -->
<script>
  let { title } = $props();           // ✅ export let → $props
  let count = $state(0);              // ✅ let → $state
  let doubled = $derived(count * 2); // ✅ $: → $derived
</script>
```

---

## Gotchas in Legacy Mode

1. **Assignment-based reactivity** - Always reassign after mutation
2. **$state in legacy** - Ignored, not an error
3. **Mixing modes** - Works but behavior can be confusing
4. **$$slots** - Compile-time feature, works normally
5. **svelte:self** - Still needs import from 'svelte'
