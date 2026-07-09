# Svelte 4 → Svelte 5 Migration Reference

## 速查对照表

| Svelte 4 | Svelte 5 | 说明 |
|---------|---------|------|
| `let count = 0` | `let count = $state(0)` | 顶层响应式 |
| `$: sum = a + b` | `let sum = $derived(a + b)` | 派生值 |
| `$: { if(c) x = v }` | `$effect(() => { if(c) x = v })` | 副作用 |
| `export let x` | `let { x } = $props()` | Props |
| `<slot />` | `{@render children()}` | 插槽 |
| `createEventDispatcher` | callback props / `$bindable` | 事件 |
| `on:click={fn}` | `onclick={fn}` | 事件监听 |
| `$$props` | `let { ...rest } = $props()` | Rest props |
| `<svelte:component this={C}` | `<svelte:element this={C}>` | 动态组件 |
| `<svelte:self>` | `import Self` | 递归组件 |

## 自动迁移脚本

```bash
# 迁移整个项目
npx svelte-migrate@latest svelte-5 ./my-project

# 迁移单个文件
npx svelte-migrate@latest svelte-5 ./src/lib/MyComponent.svelte

# 预览迁移而不执行
npx svelte-migrate@latest svelte-5 ./my-project --dry
```

## 迁移示例

### let → $state

```svelte
// Svelte 4
<script>
  let count = 0;
  function increment() { count += 1; }
</script>

// Svelte 5
<script>
  let count = $state(0);
  function increment() { count += 1; }
</script>
```

### $: → $derived / $effect

```svelte
// Svelte 4
<script>
  let a = 1, b = 2;
  $: sum = a + b;
  $: console.log('sum:', sum);
  $: {
    if (a > 10) doSomething();
  }
</script>

// Svelte 5
<script>
  let a = $state(1), b = $state(2);
  let sum = $derived(a + b);
  $effect(() => { console.log('sum:', sum); });
  $effect(() => { if (a > 10) doSomething(); });
</script>
```

### export let → $props

```svelte
// Svelte 4
<script>
  export let name: string;
  export let age = 18;
  export let items: string[] = [];
</script>

// Svelte 5
<script lang="ts">
  let {
    name,
    age = 18,
    items = []
  }: {
    name: string;
    age?: number;
    items?: string[];
  } = $props();
</script>
```

### createEventDispatcher → callback props

```svelte
// Svelte 4
<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher<{ change: string }>();
  dispatch('change', 'hello');
</script>

// Svelte 5
<script lang="ts">
  let { onChange }: { onChange?: (val: string) => void } = $props();
  onChange?.('hello');
</script>
```

### <slot> → Snippets

```svelte
// Svelte 4
<!-- Card.svelte -->
<slot name="header" />
<slot />

<!-- App.svelte -->
<Card>
  <h1 slot="header">Title</h1>
  <p>Content</p>
</Card>

// Svelte 5
<!-- Card.svelte -->
<script lang="ts">
  let {
    children,
    header
  }: {
    children: import('svelte').Snippet;
    header?: import('svelte').Snippet;
  } = $props();
</script>
{#if header}{@render header()}{/if}
{@render children()}

<!-- App.svelte -->
<Card>
  {#snippet header()}
    <h1>Title</h1>
  {/snippet}
  <p>Content</p>
</Card>
```

## Legacy Mode 强制启用

如需在 Svelte 5 中强制使用 Legacy Mode：

```svelte
<svelte:options runes={false} />

<script>
  // 使用 Svelte 4 语法
  export let name: string;
  let count = 0;
  $: doubled = count * 2;
</script>
```
