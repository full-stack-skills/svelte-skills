---
name: svelte-template-syntax
description: Svelte 5 模板语法技能。当用户需要使用 {#if}/{#each}/{#await}/{#snippet} 等块级语法、{@render}/@{html} 等模板标签、bind:、use:/transition:/in:/out:/animate:、style:/class/class:、{@attach} 等指令，或理解属性、事件、文本表达式等基础模板语法时使用。
---

# Svelte Template Syntax Reference (Svelte 5)

本技能覆盖 Svelte 5 模板语法，包括基础标记、块级表达式、模板标签、事件处理、属性绑定、style/class 指令、附件/动作、过渡和动画。

## When to use this skill

当用户需要编写或理解 Svelte 组件中的模板语法，包括条件渲染、列表渲染、异步处理、片段复用、事件绑定、双向数据绑定、过渡动画等场景时使用本技能。

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

<!-- 展开属性（顺序决定优先级：后写覆盖先写） -->
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

### await 表达式（Svelte 5.36+ experimental）

```svelte
<!-- 需 svelte.config.js 中启用 experimental.async -->
<svelte:boundary>
  <p>{await fetchData()}</p>
  {#snippet pending()}<p>loading</p>{/snippet}
</svelte:boundary>
```

特性：同步更新、并发执行（独立 `await`）、`<svelte:boundary>` pending snippet、`fork()` API。

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

Snippet 可以引用外部变量（script 变量或 `{#each}` 块级变量），并对**同一词法作用域**的兄弟/子节点可见：

```svelte
{#each items as item}
  {#snippet itemCard()}
    <div>{item.name}</div>  <!-- 引用 item -->
  {/snippet}
  {@render itemCard()}
{/each}
```

### 显式 vs 隐式 prop

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

组件标签内的非 snippet 内容自动成为 `children` snippet：

```svelte
<Button>click me</Button>

<!-- Button.svelte -->
<script>
  let { children } = $props();
</script>
<button>{@render children()}</button>
```

### 可选 snippet

```svelte
{@render children?.()}              <!-- 可选链 -->
{#if children}{:else}fallback{/if}  <!-- #if fallback -->
```

### Snippet 类型定义

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';
  let { row }: { row: Snippet<[Item]> } = $props();
</script>
```

### 导出 snippet（5.5+）

```svelte
<script module>
  export { mySnippet };
</script>
{#snippet mySnippet()}
  <div>content</div>
{/snippet}
```

### 程序化创建（createRawSnippet）

```ts
import { createRawSnippet } from 'svelte';
const greet = createRawSnippet<[string]>((name) => ({
  render: () => `<p>Hello, ${name}!</p>`
}));
```

## Critical: {@render ...}

渲染 snippet：

```svelte
{@render snippetName(args)}
{@render children?.()}  <!-- 可选链安全调用 -->
{@render (cond ? a : b)()}  <!-- 任意表达式 -->
```

## Critical: {@html ...}

插入原始 HTML（**注意 XSS 风险**）：

```svelte
{@html rawHtml}
```

- `{expression}` 自动转义 HTML 实体
- `{@html expression}` 直接插入原始 HTML，需确保内容可信
- 必须是**完整独立** HTML（不能跨 `{@html}` 拼接标签）
- 不受 scoped 样式影响 — 用 `:global` 包裹

## Critical: {@attach ...}

元素挂载时运行的函数（Svelte 5.29+），取代 `use:` action：

```svelte
<script>
  /** @type {import('svelte/attachments').Attachment} */
  function tooltip(node) {
    const tip = createTip(node);
    return { destroy() { tip.destroy(); } };
  }
</script>

<button {@attach tooltip()}>hover</button>
```

### 带参数的 Attachment Factory

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

### 内联 Attachment

```svelte
<canvas {@attach (canvas) => {
  const ctx = canvas.getContext('2d');
  $effect(() => { ctx.fillStyle = color; /* ... */ });
}}></canvas>
```

### 重新运行行为

`{@attach foo(bar)}` 在 `foo` 或 `bar` 变化时**完整重建**（与 action 不同）。

### 从 Action 转换

```js
import { fromAction } from 'svelte/attachments';
const attach = fromAction(myAction);
```

## Critical: bind: 指令

数据从子级回流到父级：

```svelte
<!-- 简写 -->
<input bind:value />
<input bind:value={value} />
```

### Function Bindings（5.9+）

```svelte
<input bind:value={
  () => value,
  (v) => value = v.toLowerCase()
} />
```

### 完整 bind: 列表

| 指令 | 元素 | 类型 |
|------|------|------|
| `bind:value` | input/textarea/select | 双向 |
| `bind:checked` | input[type=checkbox] | 双向 |
| `bind:indeterminate` | input[type=checkbox] | 双向 |
| `bind:group` | radio/checkbox 组 | 双向 |
| `bind:files` | input[type=file] | 双向 |
| `bind:open` | details | 双向 |
| `bind:value` | select multiple | 数组 |
| `bind:currentTime/paused/volume/muted/playbackRate` | audio/video | 双向 |
| `bind:duration/buffered/seeking/ended/readyState/played` | audio/video | 只读 |
| `bind:videoWidth/Height` | video | 只读 |
| `bind:naturalWidth/Height` | img | 只读 |
| `bind:innerHTML/innerText/textContent` | contenteditable | 双向 |
| `bind:clientWidth/Height/offsetWidth/Height/contentRect` | 块级元素 | 只读 |
| `bind:this` | 元素/组件 | 引用 |
| `bind:innerWidth/Height` | svelte:window | 双向 |
| `bind:scrollX/Y` | svelte:window | 双向 |
| `bind:online` | svelte:window | 双向 |
| `bind:fullscreenElement` | svelte:document | 双向 |
| `bind:visibilityState` | svelte:document | 双向 |

### $bindable Props

```svelte
<!-- 子组件 -->
<script>
  let { value = $bindable() } = $props();
</script>
<input bind:value />

<!-- 父组件 -->
<Child bind:value={text} />
```

## Critical: use: Action（Svelte 5.29 前主推，5.29+ 推荐 {@attach}）

```svelte
<script>
  /** @type {import('svelte/action').Action<HTMLElement, {color: string}>} */
  function paint(node, data) {
    node.style.color = data.color;
  }
</script>

<div use: paint={{ color: 'red' }}>red text</div>
```

Action 只调用一次（不 SSR）；参数变化**不**重新运行。

## Critical: transition: / in: / out:

元素进入/离开 DOM 时的过渡：

```svelte
<script>
  import { fade, fly } from 'svelte/transition';
  let visible = $state(false);
</script>

{#if visible}
  <div transition:fade>fades in and out</div>
  <div in:fly={{ y: 200 }} out:fade>flies in, fades out</div>
{/if}
```

- `transition:` 双向（可中途反向）
- `in:` / `out:` 单向（并行不打断）
- 默认**局部**（`|global` 修饰符响应祖先块）

### 内置 transitions

`fade` / `blur` / `fly` / `slide` / `scale` / `draw` / `crossfade`（来自 `svelte/transition`）

### 自定义 transition

```js
function whoosh(node, params) {
  return {
    duration: 400,
    easing: elasticOut,
    css: (t, u) => `transform: scale(${t})`
  };
}
```

### 过渡事件

`onintrostart` / `onintroend` / `onoutrostart` / `onoutroend`

## Critical: animate:

keyed each 块内已有元素**重排**时的动画：

```svelte
<script>
  import { flip } from 'svelte/animate';
</script>

{#each list as item, i (item)}
  <li animate:flip={{ duration: 300 }}>{item}</li>
{/each}
```

- **不**在新增/删除时触发（用 transition:）
- **必须** keyed `{#each}` 的直接子元素
- `flip` 是内置（First-Last-In-First-Out 平滑移动）

## Critical: style: / class: / class

### style: 指令

```svelte
<div style:color="red" style:width="12rem" style:--columns={n}>
  <!-- 简写、表达式、!important、CSS 自定义属性都支持 -->
</div>
```

`style:` 指令优先级**高于** `style` 属性（即使带 `!important`）。

### class 属性（5.16+ 支持对象/数组）

```svelte
<!-- 对象：truthy key 添加 -->
<div class={{ cool, lame: !cool }}>...</div>

<!-- 数组：falsy 过滤后合并 -->
<div class={[faded && 'saturate-0', large && 'scale-200']}>...</div>

<!-- 嵌套扁平化 -->
<div class={['btn', { primary: true }, ['rounded', { large }]]}>...</div>
```

### ClassValue 类型

```svelte
<script lang="ts">
  import type { ClassValue } from 'svelte/elements';
  const props: { class: ClassValue } = $props();
</script>
```

### class: 指令（传统，5.16+ 不再推荐）

```svelte
<div class:cool={cool} class:lame={!cool}>
  <!-- 简写 class:cool -->
</div>
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

仅能为 `{#if}`、`{#each}`、`{#snippet}`、`<Component />`、`<svelte:boundary>` 的直接子元素。

## Critical: {let/const ...}（5.56+ 推荐）

```svelte
{#each boxes as box}
  {const area = box.width * box.height}
  <p>{area}</p>
{/each}
```

可在组件任意位置；引用外部值，作用域为词法兄弟/子节点。

响应式：

```svelte
{let name = $state(user.name)}
{const greeting = $derived(`Hello ${name}`)}
```

## Critical: {@debug ...}

调试标签，变量变化时打印到控制台：

```svelte
{@debug user, count}       <!-- 指定变量 -->
{@debug}                   <!-- 任意状态变化断点 -->
```

**注意**：`{@debug}` 只能接受变量名（不能是表达式）。

## Quick Fixes

| 问题 | 解决方案 |
|------|----------|
| 列表不更新 | 使用带 key 的 `{#each items as item (item.id)}` |
| 空列表无 fallback | 加 `{:else}` 分支 |
| 过渡不触发 | 用 `{#key value}` 包裹触发重建 |
| `{@html}` 样式不生效 | 用 `:global` 或不用 `{@html}` |
| Snippet 不可见 | 确认在同一词法作用域内 |
| `await` 同步行为不工作 | 启用 `experimental.async` |
| 反向过渡不工作 | 用 `transition:` 而非 `in:` / `out:` |
| `animate:` 不触发 | 必须 keyed `{#each}` + 直接子元素 |

## Gotchas

1. **`{#each}` 的 key 必须是唯一值** — 使用索引（`i`）作 key 会导致不正确的高效更新
2. **`{@html}` 不受 scoped 样式影响** — 插入的 HTML 不包含 Svelte 作用域哈希
3. **`{@debug}` 不能调试表达式** — 只接受变量名
4. **Snippet 在词法作用域内可见** — 嵌套层级决定可见性
5. **`{@attach}` 是 Svelte 5.29+** — 旧版本用 `use:` action
6. **事件属性以 `on` 开头** — `onclick` 不是 `on:click`（Svelte 4 语法已废弃）
7. **`{@attach}` 重新运行完整 setup** — 与 action 不重新运行相反；昂贵 setup 用 `getBar` 形式包装
8. **`class` 属性的 `false`/`NaN` 序列化为字符串** — 5.16+ 用对象形式
9. **`bind:group` 仅在同一组件内有效** — 跨组件失效
10. **Transitions 在 SSR 期间不运行** — 客户端 hydration 后激活

## FAQ

**Q: `{#each}` 中 `key` 的作用是什么？**
A: 帮助 Svelte 高效 diff：当数据变化时，通过 key 精确识别哪个元素变化（插入/移动/删除），避免整体重渲染。

**Q: Snippet 和 Slot 有什么区别？**
A: Snippet 是 Svelte 5 的新机制，更强大灵活。Slot 只能传递标记；Snippet 可以接收参数、可在定义处直接使用、可传递到组件、词法作用域共享。

**Q: `{@html}` 和普通文本插值的区别？**
A: `{expr}` 自动 HTML 转义；`{@html expr}` 直接插入原始 HTML。原始 HTML 不受 scoped 样式影响。

**Q: 什么时候用 `{#key}`？**
A: 当需要表达式变化时触发动画/重建时，例如路由切换、用户切换等场景。

**Q: `use:` 和 `{@attach}` 怎么选？**
A: Svelte 5.29+ 推荐 `{@attach}` — 更灵活（参数响应、可用于组件）。旧库只提供 action 时用 `fromAction(act)` 包装。

**Q: `transition:` vs `in:` / `out:`？**
A: `transition:` 双向可反向；`in:` / `out:` 单向并行播放不互相打断。

**Q: `animate:` 和 `transition:` 区别？**
A: `animate:` 是 keyed each 中已有元素**重排**时；`transition:` 是元素进入/离开 DOM 时。

**Q: 什么时候用 `await` 表达式 vs `{#await}` 块？**
A: 新代码推荐 `await` 表达式（需 `experimental.async`），支持同步更新、并发。旧代码或简单分支可用 `{#await}` 块。

## Examples

可执行的代码示例，见 `examples/` 目录：

| 文件 | 内容 |
|------|------|
| `basic-markup.md` | 标签、属性、展开、文本表达式、注释、事件委托 |
| `if-await-snippet.md` | {#if}/{#each}/{#await}/{#snippet} 基础到递归 |
| `snippet-advanced.md` | Snippet 作用域、显式/隐式 prop、可选、类型化、模块导出、createRawSnippet |
| `render-tags.md` | {@render}/{@html}/{@debug}/{@const}/{@attach} |
| `attachments.md` | {@attach} factory / inline / 条件 / 转换 action |
| `use-action.md` | use: 指令 + Action 类型签名 + use→{@attach} 迁移 |
| `transitions.md` | transition:/in:/out: 参数、自定义函数、事件、crossfade |
| `animations.md` | animate:flip / 自定义函数 / 与 transition 组合 |
| `await-expressions.md` | synchronized / 并发 / pending / fork / SSR |
| `style-class.md` | style: 指令、class 对象/数组、class: 传统、ClassValue |
| `bind-directives.md` | bind:value/checked/group/files/this/offsetWidth/Function bindings |
| `event-handlers.md` | onclick、事件冒泡、委托、window/document 事件 |

## References

深入技术参考，见 `references/` 目录：

| 文件 | 内容 |
|------|------|
| `basic-markup.md` | 标签语义、属性规则、事件委托、注释、SSR 行为 |
| `if-await-each.md` | {#if}/{#else}/{#each}/{#await} 完整语法及 keyed 机制 |
| `snippet-render.md` | Snippet 定义/调用/递归/传参，@render/@const 类型签名 |
| `snippet-advanced.md` | 显式/隐式 prop、可选 snippet、createRawSnippet、模块导出、Snippet vs Slot |
| `special-tags.md` | {@html}/{@debug}/{@attach}/svelte:element |
| `attachments.md` | {@attach} 语义、reactive 重运行、fromAction 迁移、svelte/attachments API |
| `transitions.md` | 局部/全局、内置 transitions、自定义函数 css/tick、events、crossfade |
| `animations.md` | animate:flip、自定义函数（from/to DOMRect）、与 transition 组合 |
| `await-expressions.md` | 同步更新、并发、$effect.pending()、settled()、fork、SSR |
| `style-class.md` | style: 优先级、class 对象/数组/ClassValue、!important 规则 |
| `bind-reference.md` | 所有 bind: 指令完整列表及适用元素 |
| `event-reference.md` | 事件处理：onclick vs on:click、window/document 事件 |
