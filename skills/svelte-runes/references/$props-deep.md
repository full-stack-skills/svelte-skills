# $props 完整参考

## 基本解构

```svelte
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

## $bindable 可变绑定

```svelte
<!-- 父组件 -->
<script>
  let value = $state('hello');
</script>

<!-- 单向（只读）-->
<Child {value} />

<!-- 双向绑定 -->
<Child bind:value />
```

```svelte
<!-- Child.svelte -->
<script lang="ts">
  let {
    value = $bindable('')
  }: {
    value?: string;
  } = $props();
</script>

<input bind:value />
```

## Rest Props

```svelte
<script lang="ts">
  let {
    variant = 'primary',
    size = 'md',
    ...rest
  }: {
    variant?: 'primary' | 'secondary';
    size?: 'sm' | 'md' | 'lg';
    [key: string]: any;
  } = $props();
</script>

<button class="btn btn-{variant} btn-{size}" {...rest}>
  <slot />
</button>
```

## 泛型组件

```svelte
<script lang="ts" generics="T extends { id: number }">
  import type { Snippet } from 'svelte';

  let {
    items,
    renderItem
  }: {
    items: T[];
    renderItem: Snippet<[T]>;
  } = $props();
</script>

{#each items as item (item.id)}
  {@render renderItem(item)}
{/each}
```

## Snippet 作为 Props

```svelte
<!-- Modal.svelte -->
<script lang="ts">
  let {
    title,
    children
  }: {
    title: string;
    children: Snippet;
  } = $props();
</script>

<div class="modal">
  <h2>{title}</h2>
  {@render children()}
</div>
```

## 默认值规则

- `age = 18` → age 可省略，省略时用 18
- `items = $bindable([])` → 可变绑定，默认空数组
- Props 是响应式的，默认值只在省略时生效
