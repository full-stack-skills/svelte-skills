# {@render} / {@html} / {@debug} / {@const}

## 1. {@render} 调用 Snippet

```svelte
{#snippet header(title)}
  <h1>{title}</h1>
{/snippet}

{@render header('Welcome')}
{@render header('About')}
```

## 2. {@render} + children

```svelte
<!-- Modal.svelte -->
<script lang="ts">
  let {
    title,
    children
  }: { title: string; children: import('svelte').Snippet } = $props();
</script>

<div class="modal">
  <h2>{title}</h2>
  {@render children()}
</div>

<!-- Parent.svelte -->
<Modal title="Info">
  {#snippet children()}
    <p>Some content here</p>
  {/snippet}
</Modal>
```

## 3. {@render} 条件渲染

```svelte
<script>
  let show = $state(true);
</script>

{#snippet panel(content)}
  <div class="panel">{content}</div>
{/snippet}

{#if show}
  {@render panel('Content')}
{/if}
```

## 4. {@html} 渲染 HTML 字符串

```svelte
<script>
  let html = '<strong>Bold</strong> and <em>italic</em>';
</script>

{@html html}
```

⚠️ **安全警告**：只用于可信内容，XSS 风险：
```svelte
<!-- ❌ 危险：用户输入未经转义 -->
{@html userInput}

<!-- ✅ 安全：确保内容可信 -->
{@html sanitize(userInput)}
```

## 5. {@debug} 开发调试

```svelte
<script>
  let count = $state(0);
  let name = $state('Alice');
</script>

<!-- count 或 name 变化时在控制台打印 -->
{@debug count, name}

<!-- 等价于： -->
{@debug}

<p>{count} - {name}</p>
```

## 6. {@debug} 自定义标签

```svelte
{@debug count, name}

<!-- 控制台输出： -->
<!-- {count: 0, name: "Alice"} -->
```

## 7. {@const} 模板内常量

```svelte
{#each items as item}
  {@const doubled = item.price * 2}
  <p>{item.name}: ${doubled}</p>
{/each}
```

## 8. {@const} + 条件计算

```svelte
{#if showPrices}
  {#each items as item}
    {@const discount = item.price * 0.9}
    <p>
      {item.name}:
      <s>${item.price}</s>
      ${discount.toFixed(2)}
    </p>
  {/each}
{/if}
```

## 9. {@const} 在 {#each} 中

```svelte
{#each users as user}
  {@const isAdmin = user.role === 'admin'}
  <div class:admin={isAdmin}>
    {user.name}
    {#if isAdmin}
      <span>⭐</span>
    {/if}
  </div>
{/each}
```

## 10. 组合：{@render} + {@const}

```svelte
{#snippet itemCard(item)}
  {@const tag = item.featured ? 'Featured' : 'Standard'}
  <div class="card">
    <span class="tag">{tag}</span>
    <h3>{item.title}</h3>
    <p>{item.description}</p>
  </div>
{/snippet}

{#each items as item (item.id)}
  {@render itemCard(item)}
{/each}
```

## 11. {@attach}（替代 use:action）

```svelte
<script>
  import { initializeChart } from './chart.js';

  let data = $state([1, 2, 3]);
</script>

<!-- 替代 Svelte 4 的 use:initializeChart -->
<div {@attach initializeChart(data)}></div>
```
