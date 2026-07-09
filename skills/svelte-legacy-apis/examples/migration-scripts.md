# Migration Scripts - Svelte 4 to Svelte 5

This guide covers the complete migration workflow with practical examples.

## Automated Migration Tool

### Running npx svelte-migrate@latest svelte-5

```bash
# Navigate to your Svelte 4 project
cd my-svelte4-project

# Run the automated migration
npx svelte-migrate@latest svelte-5

# Preview what will be migrated (dry run)
npx svelte-migrate@latest svelte-5 --dry-run

# Migrate specific files
npx svelte-migrate@latest svelte-5 src/lib/Counter.svelte
```

The migration tool handles:
- `export let` → `$props()`
- `let x = y` (top-level reactive) → `$state`
- `$: x = y` → `$derived`
- `on:event` → `onevent`
- `class:name={bool}` → `class:name={bool}` (unchanged in Svelte 5)

### What svelte-migrate Does NOT Handle

Manual migration required for:
- `createEventDispatcher` → callback props
- `<slot>` → snippets (`{@render children()}`)
- `$: { block }` → `$effect`
- `$$props` / `$$restProps`
- `svelte:component`

---

## let → $state Migration

### Simple let (Automatic Reactive)

**Before (Svelte 4):**
```svelte
<script>
  let count = 0;
</script>
<button on:click={() => count++}>{count}</button>
```

**After (Svelte 5):**
```svelte
<script>
  let count = $state(0);
</script>
<button onclick={() => count++}>{count}</button>
```

### Object let

**Before:**
```svelte
<script>
  let user = { name: 'Alice', age: 30 };
</script>
<p>{user.name}</p>
<button on:click={() => user.age++}>Birthday!</button>
```

**After:**
```svelte
<script>
  let user = $state({ name: 'Alice', age: 30 });
</script>
<p>{user.name}</p>
<button onclick={() => user.age++}>Birthday!</button>
```

### Array let (Important Gotcha!)

**Before:**
```svelte
<script>
  let items = [1, 2, 3];

  function addItem(item) {
    items.push(item);    // ❌ Does NOT trigger update in Legacy!
    items = items;       // ✅ Workaround: reassign
  }
</script>
```

**After:**
```svelte
<script>
  let items = $state([1, 2, 3]);

  function addItem(item) {
    items.push(item);    // ✅ Works in $state!
  }
</script>
```

---

## $: → $derived Migration

### Simple Reactive Statement

**Before:**
```svelte
<script>
  let a = 1;
  let b = 2;
  $: sum = a + b;
</script>
<p>{sum}</p>
```

**After:**
```svelte
<script>
  let a = $state(1);
  let b = $state(2);
  let sum = $derived(a + b);
</script>
<p>{sum}</p>
```

### Reactive with Console.log

**Before:**
```svelte
<script>
  let count = 0;
  $: console.log(`count is ${count}`);
</script>
```

**After:**
```svelte
<script>
  let count = $state(0);
  $effect(() => console.log(`count is ${count}`));
</script>
```

### Reactive Block

**Before:**
```svelte
<script>
  let items = [];
  $: {
    total = 0;
    for (const item of items) {
      total += item.price;
    }
  }
</script>
```

**After:**
```svelte
<script>
  let items = $state([]);
  let total = $derived(() => {
    let t = 0;
    for (const item of items) {
      t += item.price;
    }
    return t;
  });
  // Or more simply:
  let total = $derived(items.reduce((sum, item) => sum + item.price, 0));
</script>
```

---

## export let → $props Migration

### Basic Props

**Before:**
```svelte
<!-- Greeter.svelte -->
<script>
  export let name;
  export let greeting = 'Hello';
</script>
<p>{greeting}, {name}!</p>
```

**After:**
```svelte
<!-- Greeter.svelte -->
<script>
  let { name, greeting = 'Hello' } = $props();
</script>
<p>{greeting}, {name}!</p>
```

### Rest Props

**Before:**
```svelte
<script>
  export let title;
  let { ...rest } = $$props;
</script>
<h1 {...rest}>{title}</h1>
```

**After:**
```svelte
<script>
  let { title, ...rest } = $props();
</script>
<h1 {...rest}>{title}</h1>
```

### API Exports (Functions/Constants)

**Before:**
```svelte
<script>
  export function greet(name) {
    alert(`Hello ${name}!`);
  }
  export const VERSION = '1.0';
</script>
```

**After - No change needed:**
```svelte
<script>
  export function greet(name) {
    alert(`Hello ${name}!`);
  }
  export const VERSION = '1.0';
</script>
```
> Note: `export` for functions/constants still works in Svelte 5.

---

## on:click → onclick Migration

### Simple Event Handler

**Before:**
```svelte
<button on:click={handleClick}>Click me</button>
```

**After:**
```svelte
<button onclick={handleClick}>Click me</button>
```

### Inline Handler

**Before:**
```svelte
<button on:click={() => count++}>Increment</button>
```

**After:**
```svelte
<button onclick={() => count++}>Increment</button>
```

### Event Modifier (e.stopPropagation)

**Before:**
```svelte
<script>
  function handleClick(e) {
    e.stopPropagation();
    doSomething();
  }
</script>
<div on:click={handleClick}>
```

**After:**
```svelte
<script>
  function handleClick(e) {
    e.stopPropagation();
    doSomething();
  }
</script>
<div onclick={handleClick}>
```

> Note: Event modifiers like `|preventDefault` are removed. Handle them manually in the handler.

### preventDefault Example

**Before:**
```svelte
<form on:submit|preventDefault={handleSubmit}>
```

**After:**
```svelte
<script>
  function handleSubmit(e) {
    e.preventDefault();
    // handle submit
  }
</script>
<form onsubmit={handleSubmit}>
```

---

## slot → {@render children()} Migration

### Default Slot

**Before:**
```svelte
<!-- Card.svelte -->
<div class="card">
  <slot />
</div>

<!-- Usage -->
<Card>
  <p>Content here</p>
</Card>
```

**After:**
```svelte
<!-- Card.svelte -->
<script>
  let { children } = $props();
</script>
<div class="card">
  {@render children()}
</div>

<!-- Usage - same -->
<Card>
  <p>Content here</p>
</Card>
```

### Named Slots

**Before:**
```svelte
<!-- Layout.svelte -->
<header><slot name="header" /></header>
<main><slot /></main>
<footer><slot name="footer" /></footer>

<!-- Usage -->
<Layout>
  <h1 slot="header">Title</h1>
  <p>Main content</p>
  <p slot="footer">Footer</p>
</Layout>
```

**After:**
```svelte
<!-- Layout.svelte -->
<script>
  let { header, children, footer } = $props();
</script>
<header>{@render header?.()}</header>
<main>{@render children()}</main>
<footer>{@render footer?.()}</footer>

<!-- Usage -->
<Layout>
  {#snippet header()}
    <h1>Title</h1>
  {/snippet}
  <p>Main content</p>
  {#snippet footer()}
    <p>Footer</p>
  {/snippet}
</Layout>
```

### Slot Props

**Before:**
```svelte
<!-- List.svelte -->
<script>
  export let items = [];
</script>
<ul>
  {#each items as item}
    <slot {item} />
  {/each}
</ul>

<!-- Usage -->
<List {items} let:item>
  <li>{item.name}</li>
</List>
```

**After:**
```svelte
<!-- List.svelte -->
<script>
  let { items = [], children } = $props();
</script>
<ul>
  {#each items as item}
    {@render children({ item })}
  {/each}
</ul>

<!-- Usage -->
<List {items}>
  {#snippet children({ item })}
    <li>{item.name}</li>
  {/snippet}
</List>
```

---

## createEventDispatcher → Callback Props Migration

### Basic Usage

**Before:**
```svelte
<!-- Button.svelte -->
<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  function handleClick() {
    dispatch('click', { timestamp: Date.now() });
  }
</script>
<button on:click={handleClick}>Click</button>

<!-- Parent.svelte -->
<Button on:click={(e) => console.log(e.detail)} />
```

**After:**
```svelte
<!-- Button.svelte -->
<script>
  let { onClick } = $props();

  function handleClick() {
    onClick?.({ timestamp: Date.now() });
  }
</script>
<button onclick={handleClick}>Click</button>

<!-- Parent.svelte -->
<Button onClick={(detail) => console.log(detail)} />
```

### Multiple Events

**Before:**
```svelte
<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  dispatch('change', { value: 42 });
  dispatch('complete');
</script>

<!-- Parent -->
<Component
  on:change={(e) => handleChange(e.detail)}
  on:complete={handleComplete}
/>
```

**After:**
```svelte
<script>
  let { onChange, onComplete } = $props();

  onChange?.({ value: 42 });
  onComplete?.();
</script>

<!-- Parent -->
<Component
  onChange={(detail) => handleChange(detail)}
  onComplete={handleComplete}
/>
```

### TypeScript Support

**Before:**
```svelte
<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher<{
    change: { value: number };
    complete: null;
  }>();
</script>
```

**After:**
```svelte
<script lang="ts">
  let { onChange, onComplete } = $props<{
    onChange?: (detail: { value: number }) => void;
    onComplete?: () => void;
  }>();
</script>
```

---

## Migration Checklist

1. [ ] Run `npx svelte-migrate@latest svelte-5` for automated parts
2. [ ] Migrate `let` → `$state`
3. [ ] Migrate `$:` → `$derived` / `$effect`
4. [ ] Migrate `export let` → `$props()`
5. [ ] Migrate `on:event` → `onevent`
6. [ ] Migrate `<slot>` → `{@render children()}`
7. [ ] Migrate `createEventDispatcher` → callback props
8. [ ] Update `$$props` / `$$restProps` usage
9. [ ] Test all components thoroughly
10. [ ] Update TypeScript types if needed
