# Svelte Legacy APIs - Examples

This directory contains practical examples for working with Legacy APIs in Svelte 5.

## Contents

- **[migration-scripts.md](./migration-scripts.md)** - Step-by-step migration workflows from Svelte 4 to Svelte 5
- **[legacy-mode.md](./legacy-mode.md)** - Using legacy mode in Svelte 5 projects
- **[store-comparison.md](./store-comparison.md)** - Store patterns in both Svelte 4 and Svelte 5

## Quick Examples

### Reactive let (Legacy) vs $state (Runes)

**Legacy (Svelte 4):**
```svelte
<script>
  let count = 0; // Automatically reactive
</script>
<button on:click={() => count++}>{count}</button>
```

**Runes (Svelte 5):**
```svelte
<script>
  let count = $state(0);
</script>
<button onclick={() => count++}>{count}</button>
```

### export let (Legacy) vs $props (Runes)

**Legacy:**
```svelte
<script>
  export let name;
  export let age = 18;
</script>
```

**Runes:**
```svelte
<script>
  let { name, age = 18 } = $props();
</script>
```

### $: reactive statement (Legacy) vs $derived (Runes)

**Legacy:**
```svelte
<script>
  let a = 1, b = 2;
  $: sum = a + b;
</script>
```

**Runes:**
```svelte
<script>
  let a = $state(1), b = $state(2);
  let sum = $derived(a + b);
</script>
```

## Navigation

For detailed migration references, see the [references](../references/README.md) directory.
