# Snippets — Advanced Reference

## 定义语法

```svelte
{#snippet name()}
  ...
{/snippet}

{#snippet name(param1, param2, paramN)}
  ...
{/snippet}
```

可：
- 有任意数量的参数
- 每个参数可设默认值
- 使用解构模式
- **不能**使用 rest 参数

## 词法作用域

Snippets 声明在组件**任意位置**，对其**同一词法作用域**的兄弟/子节点可见：

```svelte
<div>
  {#snippet x()}
    {#snippet y()}...{/snippet}
    {@render y()}  <!-- OK：y 是 x 的子 -->
  {/snippet}

  {@render y()}  <!-- 错误：y 在 div 的兄弟作用域外 -->
</div>

{@render x()}  <!-- 错误：x 在 div 外部 -->
```

可引用：
- `<script>` 顶层声明
- `{#each}` 块级变量
- 父级 snippet

可自引用 / 相互引用：

```svelte
{#snippet countdown(n)}
  {#if n > 0}
    <span>{n}...</span>
    {@render countdown(n - 1)}
  {/if}
{/snippet}
```

## 显式 vs 隐式 props

### 显式 prop

```svelte
<Table {data} {header} {row} />
```

snippet 作为 prop 值传递。

### 隐式 prop

在组件标签内直接声明 snippet，自动作为同名 prop：

```svelte
<Table {data}>
  {#snippet header()}
    <th>fruit</th>
  {/snippet}
  {#snippet row(d)}
    <td>{d.name}</td>
  {/snippet}
</Table>
```

### 隐式 children

组件标签内**非 snippet 声明**的内容自动成为 `children` snippet：

```svelte
<Button>click me</Button>

<!-- Button.svelte -->
<script>
  let { children } = $props();
</script>
<button>{@render children()}</button>
```

> 不能同时定义 `children` prop 又放非 snippet 内容 — 建议避免使用 `children` 作为 prop 名。

## 可选 snippet

两种方式：

```svelte
<!-- 1. 可选链 -->
{@render children?.()}

<!-- 2. #if 提供 fallback -->
{#if children}
  {@render children()}
{:else}
  <p>fallback content</p>
{/if}
```

## 类型签名

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

`Snippet` 类型参数是参数 tuple（不是 array）。

### 泛型约束

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

## 模块导出

顶层 snippet 可从 `<script module>` 导出：

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

约束：
- 需 Svelte 5.5.0+
- 被导出的 snippet 不能引用非 module `<script>` 中的声明（直接或通过其他 snippet）

## 程序化创建

`createRawSnippet` 高级 API：

```ts
import { createRawSnippet } from 'svelte';

const greet = createRawSnippet<[string]>((name) => {
  return {
    render: () => `<p>Hello, ${name}!</p>`,
    setup: (node) => {
      // 可选：mount 时挂载
    }
  };
});
```

## Snippet vs Slot（Svelte 4 兼容）

| 特性 | Snippet (Svelte 5) | Slot (Svelte 4) |
|------|---------------------|-----------------|
| 传参 | 任意参数 | 无 |
| 命名 | `{#snippet name()}` | `slot="name"` |
| 作用域 | 父级可见 | 父级不可见 |
| 类型签名 | `Snippet<[Args]>` | 无 |
| 互调 | 是 | 有限 |
| 父级访问变量 | 是 | 否 |

Svelte 5 中 slots 已废弃，请使用 snippets。

## 父子调用规则

- snippet 在父组件模板中定义
- 父组件 props 中接收
- 子组件用 `{@render snippetName(args)}` 调用
- 调用必须传递正确数量的参数（除非有默认值）

## 实际场景对照

| 场景 | 模式 |
|------|------|
| 通用包装 | `<Wrapper>{@render children?.()}</Wrapper>` |
| 渲染列表项 | `<List>{#snippet item(x)}<Item {x} />{/snippet}</List>` |
| 命名插槽 | `<Tabs>{#snippet tab1()}...{/snippet}</Tabs>` |
| 树形递归 | `{#snippet tree(nodes)}{#each nodes as n}{@render tree(n.children)}{/each}{/snippet}` |
| 函数式映射 | `{#snippet renderRow(d)}{d.name}{/snippet}` |

## 限制

- 不可使用 rest 参数（`...rest`）
- 参数不能有 `as` 类型断言外的运行时验证
- 模块导出不能引用非 module `<script>` 变量
- 子组件接收的 snippet 在子组件内不可见（除非通过 props 传递）
