# Snippets — Advanced Patterns

深入 Svelte 5 snippet 系统：作用域、显式/隐式 prop、可选片段、类型化、模块导出、程序化创建、与 slots 对比。

## 1. Snippet 作用域（词法可见性）

```svelte
<script>
  let { message = `it's great to see you!` } = $props();
</script>

{#snippet hello(name)}
  <p>hello {name}! {message}!</p>
{/snippet}

{@render hello('alice')}
{@render hello('bob')}
```

snippets 可引用外部变量（`<script>`、`{#each}` 块、参数等），并对**同一词法作用域**的兄弟/子节点可见。

```svelte
<div>
  {#snippet x()}
    {#snippet y()}...{/snippet}
    <!-- OK：y 在 x 内可见 -->
    {@render y()}
  {/snippet}

  <!-- 错误：y 不在 div 兄弟作用域 -->
  {@render y()}
</div>

<!-- 错误：x 不在 div 外部 -->
{@render x()}
```

## 2. 自引用与相互递归

```svelte
{#snippet blastoff()}
  <span>🚀</span>
{/snippet}

{#snippet countdown(n)}
  {#if n > 0}
    <span>{n}...</span>
    {@render countdown(n - 1)}
  {:else}
    {@render blastoff()}
  {/if}
{/snippet}

{@render countdown(10)}
```

## 3. 显式 prop 传递

```svelte
<!-- App.svelte -->
<script>
  import Table from './Table.svelte';
  const fruits = [
    { name: 'apples', qty: 5, price: 2 },
    { name: 'bananas', qty: 10, price: 1 }
  ];
</script>

{#snippet header()}
  <th>fruit</th><th>qty</th><th>price</th>
{/snippet}

{#snippet row(d)}
  <td>{d.name}</td><td>{d.qty}</td><td>{d.price}</td>
{/snippet}

<Table data={fruits} {header} {row} />
```

```svelte
<!-- Table.svelte -->
<script>
  let { data, header, row } = $props();
</script>

<table>
  {#if header}
    <thead><tr>{@render header()}</tr></thead>
  {/if}
  <tbody>
    {#each data as d}<tr>{@render row(d)}</tr>{/each}
  </tbody>
</table>
```

## 4. 隐式 prop 传递（推荐）

```svelte
<!-- App.svelte -->
<Table data={fruits}>
  {#snippet header()}
    <th>fruit</th><th>qty</th>
  {/snippet}

  {#snippet row(d)}
    <td>{d.name}</td><td>{d.qty}</td>
  {/snippet}
</Table>
```

`Table` 组件无需改变 — `<Table>` 标签内声明的 snippet 自动作为同名 prop。

## 5. 隐式 children

```svelte
<!-- App.svelte -->
<Button>click me</Button>

<!-- Button.svelte -->
<script>
  let { children } = $props();
</script>

<button>{@render children()}</button>
```

> 不能同时声明 `children` prop 又在组件标签内放非 snippet 内容 — 建议避免使用 `children` 作为 prop 名。

## 6. 可选 snippet 渲染

```svelte
<script>
  let { children } = $props();
</script>

<!-- 用可选链 -->
{@render children?.()}

<!-- 或用 #if 提供 fallback -->
{#if children}
  {@render children()}
{:else}
  <p>fallback content</p>
{/if}
```

## 7. 类型化 snippet

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    data: any[];
    children: Snippet;
    row: Snippet<[any]>;
  }

  let { data, children, row }: Props = $props();
</script>
```

## 8. 泛型约束（data 和 row 共用类型）

```svelte
<script lang="ts" generics="T">
  import type { Snippet } from 'svelte';

  let {
    data,
    children,
    row
  }: {
    data: T[];
    children: Snippet;
    row: Snippet<[T]>;
  } = $props();
</script>
```

## 9. 模块导出 snippet

```svelte
<!-- snippets.svelte -->
<script module>
  export { add };
</script>

{#snippet add(a, b)}
  {a} + {b} = {a + b}
{/snippet}
```

```svelte
<!-- App.svelte -->
<script>
  import { add } from './snippets.svelte';
</script>

{@render add(1, 2)}
```

> 需要 Svelte 5.5.0+。被导出的 snippet 不能引用非 module `<script>` 中的声明。

## 10. createRawSnippet（程序化创建）

```svelte
<script>
  import { createRawSnippet } from 'svelte';

  // 高级用例：动态生成 snippet
  const greet = createRawSnippet<[string]>((name) => {
    return {
      render: () => `<p>Hello, ${name}!</p>`
    };
  });
</script>

{@render greet('World')}
```

## 11. Snippet 互调（包装）

```svelte
{#snippet primaryButton(label)}
  <button class="primary">{label}</button>
{/snippet}

{#snippet withIcon(snippet, icon)}
  <span class="icon">{icon}</span>
  {@render snippet}
{/snippet}

{@render withIcon(primaryButton('Save'), '💾')}
```

## 12. 递归树（完整示例）

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

## 13. Snippets vs Slots（迁移对照）

| 特性 | Svelte 4 Slot | Svelte 5 Snippet |
|------|---------------|------------------|
| 传参 | 否（仅 fallback） | 是（任意参数） |
| 命名 | `slot="name"` | `{#snippet name()}` |
| 父级访问变量 | 否 | 是（词法作用域） |
| 类型签名 | 无 | `Snippet<[Args]>` |
| 传给子组件 | `<slot prop={x} />` | `{#snippet(x)}` |
