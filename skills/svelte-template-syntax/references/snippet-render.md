# Snippet & {@render} 完整参考

## Snippet 定义

```svelte
{#snippet name(param)}
  <div>{param}</div>
{/snippet}
```

位置：
- 可以在模板顶层（`<script>` 外）定义
- 可以在 `{#if}`、`{#each}` 等块内定义（作用域限制在块内）
- 顶层定义的 snippet 可在 `<script>` 中引用

## Snippet 调用

```svelte
{@render name('hello')}
```

## Snippet 作为 Props

```svelte
<!-- List.svelte -->
<script lang="ts">
  let {
    items,
    renderItem
  }: {
    items: any[];
    renderItem: Snippet<[any]>;
  } = $props();
</script>

{#each items as item}
  {@render renderItem(item)}
{/each}
```

```svelte
<!-- Parent.svelte -->
<List {items}>
  {#snippet renderItem(item)}
    <span>{item.name}</span>
  {/snippet}
</List>
```

## 递归 Snippet（树结构）

```svelte
{#snippet tree(nodes)}
  <ul>
    {#each nodes as node (node.id)}
      <li>
        {node.name}
        {#if node.children?.length}
          {@render tree(node.children)}
        {/if}
      </li>
    {/each}
  </ul>
{/snippet}

{@render tree(rootNodes)}
```

## Snippet 与 children

```svelte
<!-- Card.svelte -->
<script lang="ts">
  let {
    title,
    children
  }: {
    title: string;
    children: Snippet;
  } = $props();
</script>

<div class="card">
  <h2>{title}</h2>
  {@render children()}
</div>
```

```svelte
<Card title="Hello">
  {#snippet children()}
    <p>Content goes here</p>
  {/snippet}
</Card>
```

## {@const} 模板常量

```svelte
{#each items as item}
  {@const discount = item.price * 0.9}
  <p>
    {item.name}:
    <s>${item.price}</s>
    ${discount.toFixed(2)}
  </p>
{/each}
```

## Snippet 类型签名

```ts
import type { Snippet } from 'svelte';

// 无参数
let header: Snippet;

// 有参数
let item: Snippet<[string, number]>;
```
