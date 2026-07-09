# Migration Reference - Svelte 4 to Svelte 5

Complete reference for migrating from Legacy APIs to Runes.

---

## let → $state

### Syntax Comparison

| Aspect | Legacy (Svelte 4) | Runes (Svelte 5) |
|--------|-------------------|------------------|
| Basic | `let count = 0` | `let count = $state(0)` |
| Object | `let obj = { x: 1 }` | `let obj = $state({ x: 1 })` |
| Array | `let arr = [1, 2]` | `let arr = $state([1, 2])` |
| Mutation | `obj.x = 2` (no update!) | `obj.x = 2` (updates!) |
| Array push | `arr.push(3)` + reassign | `arr.push(3)` (works) |

### Before/After Examples

**Simple value:**
```svelte
<!-- Before -->
<script>
  let count = 0;
</script>

<!-- After -->
<script>
  let count = $state(0);
</script>
```

**Object:**
```svelte
<!-- Before -->
<script>
  let user = { name: 'Alice', age: 30 };
  function birthday() {
    user.age++;        // Doesn't trigger update!
    user = user;       // Must reassign
  }
</script>

<!-- After -->
<script>
  let user = $state({ name: 'Alice', age: 30 });
  function birthday() {
    user.age++;        // Works correctly!
  }
</script>
```

---

## $: → $derived / $effect

### Syntax Comparison

| Aspect | Legacy | Runes |
|--------|--------|-------|
| Derived value | `$: sum = a + b` | `let sum = $derived(a + b)` |
| Side effect | `$: console.log(x)` | `$effect(() => console.log(x))` |
| Reactive block | `$: { ... }` | `$effect(() => { ... })` |

### Before/After Examples

**Simple derived:**
```svelte
<!-- Before -->
<script>
  let a = 1;
  let b = 2;
  $: sum = a + b;
</script>

<!-- After -->
<script>
  let a = $state(1);
  let b = $state(2);
  let sum = $derived(a + b);
</script>
```

**Side effect:**
```svelte
<!-- Before -->
<script>
  let count = 0;
  $: console.log('count changed:', count);
</script>

<!-- After -->
<script>
  let count = $state(0);
  $effect(() => console.log('count changed:', count));
</script>
```

**Reactive block:**
```svelte
<!-- Before -->
<script>
  let items = [];
  $: {
    total = 0;
    for (const item of items) {
      total += item.price;
    }
  }
</script>

<!-- After -->
<script>
  let items = $state([]);
  let total = $derived(items.reduce((sum, item) => sum + item.price, 0));
</script>
```

---

## export let → $props

### Syntax Comparison

| Aspect | Legacy | Runes |
|--------|--------|-------|
| Basic prop | `export let name` | `let { name } = $props()` |
| Default value | `export let age = 18` | `let { age = 18 } = $props()` |
| Rest props | `let { ...rest } = $$props` | `let { ...rest } = $props()` |

### Before/After Examples

**Basic props:**
```svelte
<!-- Before -->
<script>
  export let name;
  export let age = 18;
  export let items = [];
</script>

<!-- After -->
<script>
  let { name, age = 18, items = [] } = $props();
</script>
```

**Renamed props:**
```svelte
<!-- Before -->
<script>
  export { internalName as name };
</script>

<!-- After -->
<script>
  let { name: internalName } = $props();
  // Or keep same name and destructure directly
</script>
```

**Rest props:**
```svelte
<!-- Before -->
<script>
  export let title;
  let { ...rest } = $$props;
</script>
<h1 {...rest}>{title}</h1>

<!-- After -->
<script>
  let { title, ...rest } = $props();
</script>
<h1 {...rest}>{title}</h1>
```

---

## $$props → Rest Props

### Before/After Examples

**Accessing all props:**
```svelte
<!-- Before -->
<script>
  let { foo, ...all } = $$props;
</script>

<!-- After -->
<script>
  let { foo, ...all } = $props();
</script>
```

**Passing unknown attributes:**
```svelte
<!-- Before -->
<script>
  export let variant = 'primary';
  let { ...rest } = $$props;
</script>
<button class={variant} {...rest}>Click</button>

<!-- After -->
<script>
  let { variant = 'primary', ...rest } = $props();
</script>
<button class={variant} {...rest}>Click</button>
```

---

## on:click → onclick

### Syntax Comparison

| Aspect | Legacy | Runes |
|--------|--------|-------|
| Basic | `on:click={handler}` | `onclick={handler}` |
| Inline | `on:click={() => x++}` | `onclick={() => x++}` |
| preventDefault | `on:submit\|preventDefault={fn}` | Manual in handler |
| stopPropagation | Manual in handler | Manual in handler |

### Before/After Examples

**Basic click:**
```svelte
<!-- Before -->
<button on:click={handleClick}>Click</button>

<!-- After -->
<button onclick={handleClick}>Click</button>
```

**With preventDefault:**
```svelte
<!-- Before -->
<form on:submit|preventDefault={handleSubmit}>

<!-- After -->
<script>
  function handleSubmit(e) {
    e.preventDefault();
    // handle submit
  }
</script>
<form onsubmit={handleSubmit}>
```

**With stopPropagation:**
```svelte
<!-- Before -->
<div on:click|stopPropagation={handleClick}>

<!-- After -->
<script>
  function handleClick(e) {
    e.stopPropagation();
    // handle click
  }
</script>
<div onclick={handleClick}>
```

---

## slot → Snippets

### Syntax Comparison

| Aspect | Legacy | Runes |
|--------|--------|-------|
| Default slot | `<slot />` | `{@render children()}` |
| Named slot | `<slot name="x" />` | `{@render x?.()}` |
| Slot props | `let:value` | `children({ value })` |
| Using slot | `<Child>content</Child>` | Same |

### Before/After Examples

**Default slot:**
```svelte
<!-- Before: Card.svelte -->
<div class="card">
  <slot />
</div>

<!-- After -->
<script>
  let { children } = $props();
</script>
<div class="card">
  {@render children()}
</div>
```

**Named slots:**
```svelte
<!-- Before: Layout.svelte -->
<header><slot name="header" /></header>
<main><slot /></main>
<footer><slot name="footer" /></footer>

<!-- Usage -->
<Layout>
  <h1 slot="header">Title</h1>
  <p slot="footer">Footer</p>
</Layout>

<!-- After: Layout.svelte -->
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
  {#snippet footer()}
    <p>Footer</p>
  {/snippet}
</Layout>
```

**Slot with props:**
```svelte
<!-- Before: List.svelte -->
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

<!-- After: List.svelte -->
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

## createEventDispatcher → Callback Props

### Syntax Comparison

| Aspect | Legacy | Runes |
|--------|--------|-------|
| Declaration | `const dispatch = createEventDispatcher()` | `let { onEvent } = $props()` |
| Dispatch | `dispatch('event', data)` | `onEvent?.(data)` |
| Handler | `on:event={handler}` | `onEvent={handler}` |

### Before/After Examples

**Basic:**
```svelte
<!-- Before: Button.svelte -->
<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  function click() {
    dispatch('click', { timestamp: Date.now() });
  }
</script>
<button on:click={click}>Click</button>

<!-- Usage -->
<Button on:click={(e) => console.log(e.detail)} />

<!-- After: Button.svelte -->
<script>
  let { onClick } = $props();

  function click() {
    onClick?.({ timestamp: Date.now() });
  }
</script>
<button onclick={click}>Click</button>

<!-- Usage -->
<Button onClick={(detail) => console.log(detail)} />
```

**Multiple events:**
```svelte
<!-- Before -->
<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  dispatch('update', { value: 42 });
  dispatch('complete');
</script>

<!-- After -->
<script>
  let { onUpdate, onComplete } = $props();

  onUpdate?.({ value: 42 });
  onComplete?.();
</script>
```

---

## $: Reactive Statement → $effect

### When to Use $effect

Use `$effect` when you need to:
- Perform side effects (console.log, DOM manipulation)
- React to changes with imperative code
- Set up subscriptions

### Before/After Examples

**Side effect:**
```svelte
<!-- Before -->
<script>
  let user;
  $: document.title = user?.name || 'App';
</script>

<!-- After -->
<script>
  let user = $state(null);
  $effect(() => {
    document.title = user?.name || 'App';
  });
</script>
```

**Cleanup with $:**
```svelte
<!-- Before -->
<script>
  import { onMount, onDestroy } from 'svelte';

  let interval;
  onMount(() => {
    interval = setInterval(() => count++, 1000);
  });
  onDestroy(() => clearInterval(interval));
</script>

<!-- After -->
<script>
  import { onMount } from 'svelte';

  onMount(() => {
    const interval = setInterval(() => count++, 1000);
    return () => clearInterval(interval);
  });
</script>
```

---

## svelte:component → svelte:element

### Syntax Comparison

| Aspect | Legacy | Runes |
|--------|--------|-------|
| Dynamic component | `<svelte:component this={Comp} />` | `<svelte:element this={Comp} />` |
| Conditional tag | Not possible | `<svelte:element this={condition ? 'div' : 'span'} />` |

### Before/After Examples

**Dynamic component:**
```svelte
<!-- Before -->
<script>
  import A from './A.svelte';
  import B from './B.svelte';

  let Current = A;
  $: Current = showB ? B : A;
</script>

<svelte:component this={Current} />

<!-- After -->
<script>
  import A from './A.svelte';
  import B from './B.svelte';

  let Current = $state(A);
  $effect(() => {
    Current = showB ? B : A;
  });
</script>

<svelte:element this={Current} />
```

---

## Summary Migration Checklist

- [ ] `let` → `$state`
- [ ] `$:` derived → `$derived`
- [ ] `$:` side effect → `$effect`
- [ ] `export let` → `$props`
- [ ] `$$props` → rest props in `$props`
- [ ] `on:event` → `onevent`
- [ ] `|preventDefault` → manual `e.preventDefault()`
- [ ] `<slot>` → `{@render children()}`
- [ ] Named slots → snippets
- [ ] `createEventDispatcher` → callback props
- [ ] `<svelte:component>` → `<svelte:element>`
- [ ] `<svelte:self>` → import and use directly
