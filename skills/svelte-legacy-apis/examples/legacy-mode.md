# Legacy Mode Examples

Legacy Mode allows Svelte 5 to run Svelte 4-style code. This is useful for gradual migration.

## Forcing Legacy Mode with runes={false}

### Component-Level Legacy

```svelte
<!-- LegacyComponent.svelte -->
<svelte:options runes={false} />

<script>
  let count = 0;  // Svelte 4 style - automatically reactive
  $: doubled = count * 2;
</script>

<button on:click={() => count++}>
  {count} (doubled: {doubled})
</button>
```

### When to Use runes={false}

Use Legacy Mode when:
- Migrating a large Svelte 4 codebase gradually
- A component depends heavily on Svelte 4 patterns
- You need interop with existing Svelte 4 components

### Project-Wide Legacy (svelte.config.js)

For an entire project to use legacy behavior:

```js
// svelte.config.js
export default {
  compilerOptions: {
    runes: false
  }
};
```

---

## Gradual Migration Strategy

### Mixing Legacy and Runes Components

Svelte 5 allows mixing legacy and runes components seamlessly:

```svelte
<!-- LegacyWrapper.svelte -->
<svelte:options runes={false} />

<script>
  let items = ['a', 'b', 'c'];
</script>

<slot />
```

```svelte
<!-- NewChild.svelte -->
<script>
  let { children } = $props();
</script>

<div class="new-child">
  {@render children?.()}
</div>
```

```svelte
<!-- App.svelte -->
<script>
  import LegacyWrapper from './LegacyWrapper.svelte';
  import NewChild from './NewChild.svelte';
  let count = $state(0);
</script>

<!-- Mix freely - they work together! -->
<LegacyWrapper>
  <NewChild>
    <p>Count: {count}</p>
  </NewChild>
</LegacyWrapper>
```

### Component Communication Between Modes

**Legacy Child with Runes Parent:**

```svelte
<!-- LegacyChild.svelte -->
<svelte:options runes={false} />
<script>
  export let name;
  let value = $state(0);  // Can use $state even in legacy mode component
</script>
<p>{name}: {value}</p>
<button on:click={() => value++}>Increment</button>
```

```svelte
<!-- RunesParent.svelte -->
<script>
  import LegacyChild from './LegacyChild.svelte';
  let initialName = $state('Counter');
</script>

<LegacyChild name={initialName} />
```

**Runes Child with Legacy Parent:**

```svelte
<!-- RunesChild.svelte -->
<script>
  let { name, onUpdate } = $props();
  let count = $state(0);
</script>
<p>{name}: {count}</p>
<button onclick={() => {
  count++;
  onUpdate?.(count);
}}>Increment</button>
```

```svelte
<!-- LegacyParent.svelte -->
<svelte:options runes={false} />
<script>
  import RunesChild from './RunesChild.svelte';
  let total = 0;
</script>

<RunesChild
  name="Counter"
  onUpdate={(val) => total = val}
/>
<p>Total: {total}</p>
```

---

## Legacy Behavior Differences

### What Works the Same

| Feature | Legacy Mode | Notes |
|---------|------------|-------|
| `export let` | ✅ Works | Props declaration |
| `let` (top-level) | ✅ Reactive | No `$state` needed |
| `$:` | ✅ Works | Reactive statements |
| `on:event` | ✅ Works | Event handlers |
| `<slot>` | ✅ Works | Content projection |
| `createEventDispatcher` | ✅ Works | Event dispatching |
| Stores | ✅ Works | `$store` auto-subscribe |

### What Doesn't Work in Legacy

| Feature | Behavior | Migration |
|---------|----------|-----------|
| `$state` | Ignored | Use `let` |
| `$derived` | Compile error | Use `$:` |
| `$effect` | Compile error | Use `$:` |
| `$props` | Compile error | Use `export let` |
| `{@render}` | Compile error | Use `<slot>` |

### What Changes in Legacy

#### Array/Object Mutation (Critical!)

In Legacy mode, assignments to properties don't trigger reactivity:

```svelte
<svelte:options runes={false} />
<script>
  let obj = { count: 0 };
  let arr = [1, 2, 3];

  function incrementObj() {
    obj.count++;  // ❌ Doesn't trigger update!
    obj = obj;    // ✅ Workaround: reassign
  }

  function addToArr() {
    arr.push(4);  // ❌ Doesn't trigger update!
    arr = arr;    // ✅ Workaround: reassign
  }
</script>
```

In Runes mode (`$state`), mutation works directly:

```svelte
<script>
  let obj = $state({ count: 0 });
  let arr = $state([1, 2, 3]);

  function incrementObj() {
    obj.count++;  // ✅ Works!
  }

  function addToArr() {
    arr.push(4); // ✅ Works!
  }
</script>
```

---

## Store Usage in Both Modes

### In Legacy Mode

```svelte
<svelte:options runes={false} />
<script>
  import { writable } from 'svelte/store';

  const count = writable(0);
  // $count auto-subscribes
</script>

<p>Count: {$count}</p>
<button on:click={() => $count++}>Increment</button>
```

### In Runes Mode

```svelte
<script>
  import { writable } from 'svelte/store';

  const count = writable(0);
</script>

<p>Count: {$count}</p>
<button onclick={() => $count++}>Increment</button>
```

> **Important:** Stores and `$store` auto-subscribe syntax work identically in both modes!

---

## Practical Migration Example

### Phase 1: Add runes={false} to stabilize

```svelte
<!-- Before: Svelte 4 -->
<script>
  export let items = [];
  $: total = items.reduce((sum, i) => sum + i, 0);
</script>
```

```svelte
<!-- Phase 1: Explicit legacy mode -->
<svelte:options runes={false} />
<script>
  export let items = [];
  $: total = items.reduce((sum, i) => sum + i, 0);
</script>
```

### Phase 2: Migrate props

```svelte
<!-- Phase 2: Use $props -->
<svelte:options runes={false} />
<script>
  let { items = [] } = $props();
  $: total = items.reduce((sum, i) => sum + i, 0);
</script>
```

### Phase 3: Migrate reactive statements

```svelte
<!-- Phase 3: Remove runes={false}, use $derived -->
<script>
  let { items = [] } = $props();
  let total = $derived(items.reduce((sum, i) => sum + i, 0));
</script>
```

---

## Best Practices for Legacy Mode

1. **Use sparingly** - Only for gradual migration, not new code
2. **Isolate legacy components** - Keep them separate from runes components when possible
3. **Document behavior** - Add comments explaining why legacy mode is used
4. **Plan migration path** - Set timeline for migrating each legacy component
5. **Test thoroughly** - Legacy and runes components interact in complex ways

### When NOT to Use Legacy Mode

- New projects or components
- When you control the component fully and can migrate
- Performance-critical code (runes are more optimized)
- When you need features only available in runes
