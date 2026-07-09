---
name: svelte-template-syntax
description: Svelte 5 模板语法技能。当用户需要使用 {#if}/{#each}/{#await}/{#snippet} 等块级语法、{@render}/@{html} 等模板标签、或理解属性、事件、文本表达式等基础模板语法时使用。
---

# Svelte Template Syntax Reference (Svelte 5)

本技能覆盖 Svelte 5 模板语法，包括块级表达式、模板标签、事件处理和文本表达式。

## When to use this skill

当用户需要编写或理解 Svelte 组件中的模板语法，包括条件渲染、列表渲染、异步处理、片段复用、事件绑定等场景时使用本技能。

## Critical: Basic Markup

### 元素标签

- 小写标签（`<div>`）= HTML 元素
- 大写/点号标签（`<Widget>`、`<my.stuff>`）= 组件

### 属性绑定

```svelte
<!-- 布尔属性 -->
<button disabled={!clickable}>...</button>
<input required={false} />

<!-- 简写（同名字相同） -->
<button {disabled}>...</button>

<!-- 展开属性 -->
<Widget {...props} a="b" />

<!-- 布尔属性：truthy 包含，falsy 排除 -->
<div data-active={isActive}>  <!-- true → 包含，false/null/undefined → 排除 -->
```

### 事件处理

```svelte
<!-- onclick 是属性，不是指令 -->
<button onclick={() => count++}>+</button>
<input {onkeydown} />          <!-- 简写形式 -->
<button {...handlerProps}>     <!-- 展开形式 -->

<!-- 事件委托：大多数事件委托到根节点，无需 stopPropagation -->
<!-- 若需阻止：使用 svelte/events 的 on 函数 -->
```

### 文本表达式

```svelte
<p>{expression}</p>           <!-- null/undefined → 省略，其余转字符串 -->
<p>Hello {name}!</p>

<!-- 原始 HTML（注意 XSS） -->
{@html rawContent}

<!-- 转义花括号 -->
<p>使用 &lbrace; 和 &rbrace;</p>
```

## Critical: {#if ...}

条件渲染：

```svelte
{#if count > 10}
  <p>big</p>
{:else if count > 5}
  <p>medium</p>
{:else}
  <p>small</p>
{/if}
```

`{:else if}` 可以链式多次，`{:else}` 为最终 fallback。

## Critical: {#each ...}

列表渲染：

```svelte
{#each items as item}
  <li>{item.name}</li>
{/each}

<!-- 带索引 -->
{#each items as item, i}
  <li>{i + 1}: {item.name}</li>
{/each}

<!-- 空列表 fallback -->
{#each todos as todo}
  <p>{todo.text}</p>
{:else}
  <p>No tasks!</p>
{/each}
```

### 带 key 的高效更新

```svelte
{#each items as item (item.id)}
  <li>{item.name}</li>
{/each}
```

Key 必须是唯一标识（字符串/数字），用于 Svelte 高效 diff 和更新 DOM（插入/移动/删除而非整体重渲染）。

### 解构和 rest

```svelte
{#each items as { id, name, ...rest }}
  <li><span>{id}</span><MyComponent {...rest} /></li>
{/each}

{#each items as [id, ...values]}
  <li><span>{id}</span></li>
{/each}
```

### 渲染 N 次

```svelte
{#each { length: 8 }, rank}
  <div class:black={(rank) % 2 === 1}></div>
{/each}
```

## Critical: {#key ...}

当表达式变化时销毁并重建内容（触发过渡动画）：

```svelte
{#key value}
  <div transition:fade>{value}</div>
{/key}
```

## Critical: {#await ...}

Promise 异步状态分支：

```svelte
{#await promise}
  <p>loading...</p>
{:then value}
  <p>result: {value}</p>
{:catch error}
  <p>error: {error.message}</p>
{/await}

<!-- 简洁形式（无 pending UI） -->
{#await promise then value}
  <p>{value}</p>
{/await}

<!-- 仅错误处理 -->
{#await promise catch error}
  <p>{error.message}</p>
{/await}
```

### 懒加载组件

```svelte
{#await import('./Heavy.svelte') then { default: Component }}
  <Component />
{/await}
```

### 同步更新（Svelte 5.36+）

`await` 表达式中的 `{#await ...}` 支持 `experimental.async` 选项（需在 `svelte.config.js` 启用）。

## Critical: {#snippet ...}

可复用的标记片段，取代 Svelte 4 的 Slots：

```svelte
{#snippet card(item)}
  <div class="card">
    <h3>{item.title}</h3>
    <p>{item.body}</p>
  </div>
{/snippet}

{@render card(item)}
```

### Snippet 作用域

Snippet 可以引用外部变量（script 变量或 `{#each}` 块级变量）：

```svelte
{#each items as item}
  {#snippet itemCard()}
    <div>{item.name}</div>  <!-- 引用 item -->
  {/snippet}
  {@render itemCard()}
{/each}
```

### 传递给组件

```svelte
<!-- 隐式 prop（推荐）-->
<Table {data}>
  {#snippet header()}
    <th>Name</th>
  {/snippet}
</Table>

<!-- 显式 prop -->
<Table {data} header={myHeader} />
```

### 隐式 children

组件标签内的内容自动成为 `children` snippet：

```svelte
<!-- App: -->
<Button>click me</Button>

<!-- Button.svelte: -->
<script>
  let { children } = $props();
</script>
<button>{@render children()}</button>
```

### Snippet 类型定义

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';
  let { row }: { row: Snippet<[Item]> } = $props();
</script>
```

### 导出 snippet

顶层 snippet 可从 `<script module>` 导出供其他组件使用：

```svelte
<script module>
  export { mySnippet };
</script>
{#snippet mySnippet()}
  <div>content</div>
{/snippet}
```

## Critical: {@render ...}

渲染 snippet：

```svelte
{@render snippetName(args)}
{@render children?.()}  <!-- 可选链安全调用 -->
```

## Critical: {@html ...}

插入原始 HTML（**注意 XSS 风险**）：

```svelte
{@html rawHtml}
```

- `{expression}` 自动转义 HTML 实体
- `{@html expression}` 直接插入原始 HTML，需确保内容可信
- 不受 scoped 样式影响

## Critical: {@attach ...}

元素挂载时运行的函数（Svelte 5.29+），取代 `use:` action：

```svelte
<script>
  function tooltip(node) {
    const tip = createTip(node);
    return { destroy() { tip.destroy(); } };
  }
</script>

<button {@attach tooltip()}>hover</button>
```

### 带参数的 Attachment

```svelte
function tooltip(content) {
  return (node) => {
    const tip = createTip(node, { content });
    return { destroy() { tip.destroy(); } };
  };
}

<button {@attach tooltip(content)}>hover</button>
```

### 条件 Attachment

```svelte
<div {@attach enabled && myAttachment}>...</div>
```

### 将 Action 转换为 Attachment

```js
import { fromAction } from 'svelte/attachments';
const attach = fromAction(myAction);
```

## Critical: {@const ...}

> ⚠️ Legacy 语法，建议用 `{const x = $derived(y)}` 替代

在块级作用域内定义常量：

```svelte
{#each boxes as box}
  {@const area = box.width * box.height}
  <p>{box.width} × {box.height} = {area}</p>
{/each}
```

仅能为 `{#if}`、`{#each}`、`{#snippet}`、`svelte:boundary` 的直接子元素。

## Critical: {@debug ...}

调试标签，变量变化时打印到控制台：

```svelte
{@debug user, count}       <!-- 指定变量 -->
{@debug}                   <!-- 任意状态变化断点 -->
```

**注意**：`{@debug}` 只能接受变量名（不能是表达式）。

## Critical: {let/const ...}

声明标签，在模板中定义局部常量：

```svelte
{#each boxes as box}
  {const area = box.width * box.height}
  <p>{area}</p>
{/each}
```

等价格式：`{const x = $derived(expr)}`（Svelte 5 推荐）。

## Quick Fixes

| 问题 | 解决方案 |
|------|----------|
| 列表不更新 | 使用带 key 的 `{#each items as item (item.id)}` |
| 空列表无 fallback | 加 `{:else}` 分支 |
| 过渡不触发 | 用 `{#key value}` 包裹触发重建 |
| `{@html}` 样式不生效 | 用 `:global` 或不用 `{@html}` |
| Snippet 不可见 | 确认在同一词法作用域内 |
| `await` 不工作 | 确认启用了 `experimental.async` |

## Gotchas

1. **`{#each}` 的 key 必须是唯一值** — 使用索引（`i`）作 key 会导致不正确的高效更新
2. **`{@html}` 不受 scoped 样式影响** — 插入的 HTML 不包含 Svelte 作用域哈希
3. **`{@debug}` 不能调试表达式** — 只接受变量名
4. **Snippet 在词法作用域内可见** — 嵌套层级决定可见性
5. **`{@attach}` 是 Svelte 5.29+** — 旧版本用 `use:` action
6. **事件属性以 `on` 开头** — `onclick` 不是 `on:click`（Svelte 4 语法已废弃）

## FAQ

**Q: `{#each}` 中 `key` 的作用是什么？**
A: 帮助 Svelte 高效 diff：当数据变化时，通过 key 精确识别哪个元素变化（插入/移动/删除），避免整体重渲染。

**Q: Snippet 和 Slot 有什么区别？**
A: Snippet 是 Svelte 5 的新机制，更强大灵活。Slot 只能传递标记；Snippet 可以接收参数、可在定义处直接使用、可传递到组件。

**Q: `{@html}` 和普通文本插值的区别？**
A: `{expr}` 自动 HTML 转义；`{@html expr}` 直接插入原始 HTML。原始 HTML 不受 scoped 样式影响。

**Q: 什么时候用 `{#key}`？**
A: 当需要表达式变化时触发动画/重建时，例如路由切换、用户切换等场景。

## Examples

可执行的代码示例，见 `examples/` 目录：

| 文件 | 内容 |
|------|------|
| `if-await-snippet.md` | {#if}/{#each}/{#await}/{#snippet} 基础到递归 |
| `bind-directives.md` | bind:value/checked/group/files/this/offsetWidth 等 |
| `event-handlers.md` | onclick、事件冒泡、委托、window/document 事件 |
| `render-tags.md` | {@render}/{@html}/{@debug}/{@const}/{@attach} |

## References

深入技术参考，见 `references/` 目录：

| 文件 | 内容 |
|------|------|
| `if-await-each.md` | {#if}/{#each}/{#await}/{:else}/{#key} 完整语法 |
| `snippet-render.md` | Snippet 定义/调用/递归、@render/@const 类型签名 |
| `bind-reference.md` | 所有 bind: 指令完整列表及适用元素 |
| `event-reference.md` | onclick vs on:click、window/document、事件委托 |
| `special-tags.md` | {@html}/{@debug}/{@attach}/svelte:element |
