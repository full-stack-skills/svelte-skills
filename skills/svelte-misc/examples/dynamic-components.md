# Dynamic Components Examples

Demonstrates `svelte:element` and `{#key}` blocks for dynamic rendering in Svelte 5.

## 1. Basic `svelte:element` with Tag Variable

Dynamic tag switching based on state:

```svelte
<script>
  let { level = 'h1' } = $props();
  const levels = ['h1', 'h2', 'h3', 'h4', 'h5', 'h6'];
</script>

<svelte:element this={level}>
  <slot />
</svelte:element>
```

```svelte
<!-- Usage -->
<DynamicHeading level="h2">Hello World</DynamicHeading>
```

---

## 2. `svelte:element` with Component Variable

Dynamically render different components:

```svelte
<script>
  import Bold from './Bold.svelte';
  import Italic from './Italic.svelte';
  import Underline from './Underline.svelte';

  let { format = 'bold' } = $props();

  const components = {
    bold: Bold,
    italic: Italic,
    underline: Underline
  };
</script>

<svelte:element this={components[format]}>
  <slot />
</svelte:element>
```

---

## 3. `{#key}` Block for Remounting

Forces component to destroy and recreate when value changes:

```svelte
<script>
  let { userId } = $props();
</script>

{#key userId}
  <UserProfile {userId} />
{/key}
```

Use case: Reset a form when switching between different users.

---

## 4. `{#key}` with Transition

Combining key blocks with transitions:

```svelte
<script>
  import { fade } from 'svelte/transition';

  let { page } = $props();
</script>

{#key page}
  <div in:fade={{ duration: 300 }} out:fade={{ duration: 200 }}>
    <svelte:component this={pages[page]} />
  </div>
{/key}
```

---

## 5. Dynamic Tag Switching (h1/h2/h3)

Real-world heading component:

```svelte
<script>
  let { tag = 'h1', children } = $props();

  const validTags = ['h1', 'h2', 'h3', 'h4', 'h5', 'h6'];
</script>

<svelte:element this={validTags.includes(tag) ? tag : 'h1'} class="heading">
  {@render children?.()}
</svelte:element>

<style>
  .heading {
    color: #333;
    margin: 0.5em 0;
  }
</style>
```

---

## 6. Dynamic Component with Props

Passing props to dynamically rendered components:

```svelte
<script>
  import Chart from './Chart.svelte';
  import Table from './Table.svelte';
  import List from './List.svelte';

  let { viewType = 'chart', data = [] } = $props();

  const views = {
    chart: Chart,
    table: Table,
    list: List
  };

  const Component = $derived(views[viewType]);
</script>

<svelte:element this={Component} {data} />
```

---

## 7. `{#key}` for Form Reset

Complete form reset example:

```svelte
<script>
  let { userId } = $props();
</script>

{#key userId}
  <form method="post" in:fade>
    <input name="name" placeholder="Name" />
    <input name="email" placeholder="Email" />
    <button type="submit">Submit</button>
  </form>
{/key}
```

---

## 8. Multiple Dynamic Tags

```svelte
<script>
  let { container = 'div', item = 'span' } = $props();
</script>

<svelte:element this={container} class="container">
  {#each items as item}
    <svelte:element this={item}>{item}</svelte:element>
  {/each}
</svelte:element>
```
