---
name: svelte-special-elements
description: Svelte 5 特殊元素技能。当用户需要使用 svelte:boundary、svelte:window、svelte:head、svelte:element、svelte:options 等特殊元素，或配置编译器选项、自定义元素、捕获渲染错误/异步 pending 时使用。
---

# Svelte Special Elements Reference (Svelte 5)

本技能覆盖 Svelte 模板中的特殊元素（以 `svelte:` 为前缀），用于处理错误边界、异步加载状态、窗口/文档/事件监听、动态标签、head 注入、编译器选项等场景。

## When to use this skill

当用户需要捕获渲染错误或异步 pending 状态（`<svelte:boundary>`）、监听 `window` / `document` / `body` 事件、动态渲染 HTML 标签（`<svelte:element>`）、将内容注入到 `document.head`（`<svelte:head>`）、或配置编译器选项/自定义元素（`<svelte:options>`）时使用本技能。

---

## Critical: `<svelte:boundary>` (Svelte 5.3+)

错误和异步边界，"隔离"应用局部以处理错误和 pending 状态。边界会捕获渲染过程中抛出的同步错误、`$effect` 中的错误、以及 `await` 表达式的 rejection；事件处理器、`setTimeout`、事件触发的异步工作等不在渲染流程中的错误**不会被**捕获。

### 三个关键属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `pending` | `Snippet` | 初次渲染时显示，直到所有 `await` 解析完成（**只**在初次渲染显示） |
| `failed` | `Snippet<[error, reset]>` | 发生错误时显示，接收 `error` 和可恢复的 `reset` 函数 |
| `onerror` | `(error, reset) => void` | 出错时调用，常用于上报到 Sentry 等；与 `failed` 并行触发 |

```svelte
<svelte:boundary onerror={(e) => report(e)}>
  <FlakyComponent />

  {#snippet pending()}
    <p>加载中…</p>
  {/snippet}

  {#snippet failed(error, reset)}
    <button onclick={reset}>oops! try again</button>
  {/snippet}
</svelte:boundary>
```

### SSR: `transformError` (Svelte 5.51+)

默认情况下，错误边界在服务端**无效**——一旦渲染出错，整个 `render(...)` 调用失败。从 5.51 起，可以在 `render(...)` / `mount(...)` / `hydrate(...)` 传入 `transformError` 函数，它必须返回一个 **JSON-stringifiable** 对象，用于在 `failed` 片段中渲染（SSR 时序列化到 HTML，客户端反序列化后用于水合）。

```js
import { render } from 'svelte/server';
const { head, body } = await render(App, {
  transformError: (error) => {
    console.error(error); // 保留原始错误用于日志
    return { message: 'An error occurred!' }; // 返回脱敏后的安全对象
  }
});
```

### 在 `onerror` 中重新抛出

如果在 `onerror` 里抛错（或重新抛出原始错误），错误会被**外层**边界捕获——可用于在保留当前 `failed` UI 的同时把错误上报到全局边界。

### `reset` 模式

`reset` 是个普通函数——可以存到 state 中，从任意位置调用（包括边界外的全局 UI、键盘快捷键等）：

```svelte
<script>
  let error = $state(null);
  let reset = $state(() => {});
  function onerror(e, r) { error = e; reset = r; }
</script>

<svelte:boundary {onerror}>
  <FlakyComponent />
</svelte:boundary>

{#if error}
  <button onclick={() => { error = null; reset(); }}>oops! try again</button>
{/if}
```

> 完整示例与参考：[examples/boundary-examples.md](./examples/boundary-examples.md)、[references/boundary-reference.md](./references/boundary-reference.md)

---

## Critical: `<svelte:window>`

监听 window 事件和绑定 window 属性：

```svelte
<svelte:window onkeydown={handleKey} onresize={handleResize} />
<svelte:window bind:scrollX bind:scrollY bind:innerWidth bind:innerHeight />
```

**可绑定属性**：

| 属性 | 类型 | 只读 | 说明 |
|------|------|------|------|
| `innerWidth` / `innerHeight` | `number` | 是 | 视口宽/高（CSS 像素） |
| `outerWidth` / `outerHeight` | `number` | 是 | 浏览器窗口外尺寸（含 chrome） |
| `scrollX` / `scrollY` | `number` | **否** | 滚动位置（**唯一可写**的窗口绑定） |
| `online` | `boolean` | 是 | `navigator.onLine` 的别名 |
| `devicePixelRatio` | `number` | 是 | 当前显示器的 device pixel ratio |

**可监听事件**：所有 window 级别事件——键盘（`onkeydown` / `onkeyup`）、鼠标（`onclick` / `onmousemove` 等）、触摸（`ontouchstart` / `ontouchmove`）、指针（`onpointerdown` 等）、滚轮（`onwheel`）、剪贴板（`oncopy` / `oncut` / `onpaste`）、拖拽（`ondrag` 系列）、焦点（`onfocus` / `onblur`）、资源（`onload` / `onerror` / `onscroll` / `onresize`）等。

> `<svelte:window>` 只能出现在组件顶层，不能在块级元素或条件块内。
>
> 初始挂载时**不会**将页面滚动到 `scrollX` / `scrollY` 的初始值（出于无障碍考虑）；只有后续值变化才会触发滚动。如需挂载即滚动，在 `$effect` 中调用 `scrollTo()`。

---

## Critical: `<svelte:document>`

监听 document 级别事件（`window` 不支持的事件）：

```svelte
<svelte:document onvisibilitychange={handleVisibility} />
<svelte:document {@attach myAttachment} />
```

**可绑定属性**（**全部 readonly**）：

| 属性 | 类型 | 说明 |
|------|------|------|
| `activeElement` | `Element \| null` | 当前焦点元素 |
| `fullscreenElement` | `Element \| null` | 当前全屏元素 |
| `pointerLockElement` | `Element \| null` | 指针锁定的元素 |
| `visibilityState` | `'visible' \| 'hidden'` | 文档可见性 |

**最常用事件**：`onvisibilitychange`（标签页切换）、`onselectionchange`（文本选择变化）、`onreadystatechange`、`onfullscreenchange`、`onpointerlockchange`、`oncopy` / `oncut` / `onpaste` 等。

> `<svelte:document>` 也只允许出现在顶层；支持 `{@attach ...}` 给 `document` 附加自定义行为。

---

## Critical: `<svelte:body>`

监听 body 元素事件（如 `mouseenter`、`mouseleave`，这些事件不在 window 上触发）：

```svelte
<svelte:body onmouseenter={handleMouseenter} onmouseleave={handleMouseleave} use:someAction />
```

> `<svelte:body>` 同样支持 `use:` action（这是给 `<body>` 加 action 的唯一干净方式）。也只允许出现在顶层。

---

## Critical: `<svelte:head>`

向 `document.head` 插入内容；SSR 时**单独暴露**于 body 之外，框架会把 head 内容放进 HTML `<head>` 标签。

```svelte
<svelte:head>
  <title>Hello world!</title>
  <meta name="description" content="SEO description" />
  <meta property="og:title" content="OG title" />
  <link rel="canonical" href="/current-url" />
</svelte:head>
```

**支持元素**：

| 元素 | 用途 |
|------|------|
| `<title>` | 文档标题 |
| `<meta>` | SEO（`name="description"`）、Open Graph（`property="og:..."`）、Twitter Card、robots 指令 |
| `<link rel="stylesheet" />` | 按页/按主题样式表 |
| `<link rel="canonical" />` | 规范 URL |
| `<link rel="prefetch" \| rel="preload" />` | 资源预取/预加载 |
| `<style>` | 内联关键 CSS |
| `<script type="application/ld+json">` | JSON-LD 结构化数据 |

**关键行为**：

- **响应式**：`$state` / `$derived` 变化时 head 自动更新。
- **重复合并**：同种元素（如两个 `<title>`）后渲染的会覆盖前一个（不重复追加）。
- **多个块**：同一组件内可有多个 `<svelte:head>`，全部合并到 head。
- **SSR 单独暴露**：`render()` 返回 `{ head, body }`，框架需要把 `head` 放进 `<head>` 标签。

> 完整示例：[examples/svelte-head-examples.md](./examples/svelte-head-examples.md)、[references/svelte-head-reference.md](./references/svelte-head-reference.md)

---

## Critical: `<svelte:element>`

动态渲染未知标签（如来自 CMS 或数据库）：

```svelte
<script>
  let tag = $state('hr');
</script>

<svelte:element this={tag}>内容</svelte:element>
<svelte:element this={tag} xmlns="http://www.w3.org/2000/svg" />
```

**行为规则**：

- `this` 为 `null` / `undefined` → 元素不渲染
- `this` 为 void 元素（`br`、`hr`、`img` 等）但有子元素 → 开发模式下运行时错误
- Svelte 自动推断 namespace（svg、mathml），可用 `xmlns` 显式指定
- `this` 必须是合法的 DOM 标签名（如 `div` / `circle`）；`#text`、`svelte:head` 等无效

**唯一支持的绑定**：`bind:this`（因为通用元素不支持 Svelte 内建绑定如 `bind:value`）

---

## Critical: `<svelte:options>`

设置编译器选项：

```svelte
<svelte:options runes={true} />
<svelte:options namespace="svg" />
<svelte:options customElement="my-element" />
<svelte:options css="injected" />
```

| 选项 | 类型 | 说明 |
|------|------|------|
| `runes={true\|false}` | `boolean` | 强制进入/退出 Runes Mode |
| `namespace="html\|svg\|mathml"` | `string` | 组件命名空间（默认 `html`） |
| `customElement={...}` | `string \| object` | 编译为自定义元素（字符串即 `tag`） |
| `css="injected"` | `string` | 样式内联注入（SSR → style 标签；CSR → JS） |

> `<svelte:options>` 必须放在 `<script>` 之后、模板之前。

**Legacy（已弃用）选项**（在 Runes 模式下无效）：`immutable={true|false}`、`accessors={true|false}`。

---

## Quick Fixes

| 问题 | 解决方案 |
|------|----------|
| 想监听 `visibilitychange` / `selectionchange` | 用 `<svelte:document>` 而非 `<svelte:window>` |
| 想监听 `mouseenter` / `mouseleave` | 用 `<svelte:body>` |
| 动态标签不渲染 | 检查 `this` 是否为 nullish |
| 样式不生效 | 用 `css="injected"` 强制内联 |
| 渲染抛错整个应用挂掉 | 用 `<svelte:boundary>` + `failed` 片段隔离 |
| 异步加载时显示骨架屏 | 用 `<svelte:boundary>` + `pending` 片段（**仅**首次渲染） |
| 渲染时想上报到 Sentry | 用 `<svelte:boundary onerror={...}>` |
| 想让 `reset` 在边界外触发 | 把 `error` / `reset` 存到 `$state` |
| 动态改 `<title>` 或 `<meta>` | 用 `<svelte:head>` 包裹响应式表达式 |

---

## Gotchas

1. **只能出现在顶层** — `svelte:window/document/body/head/element` 不能在 `{#if}` 或其他块级元素内。
2. **SSR 行为不同** — `<svelte:head>` 在 SSR 时内容单独暴露，不在 body 内；`<svelte:boundary>` 默认对 SSR 无效（5.51+ 起可用 `transformError` 启用）。
3. **`svelte:element` 是通用绑定** — 不支持 `bind:value` 等元素特有绑定。
4. **`svelte:window` 的 `scrollX` / `scrollY` 是唯一可写绑定** — 其他都 readonly。
5. **`<svelte:boundary>` 不捕获事件处理器 / `setTimeout` 中的错误** — 这些不是渲染流程。
6. **`<svelte:boundary>` 的 `pending` 只在初次渲染显示** — 后续异步更新用 `$effect.pending()`。

---

## FAQ

**Q: `<svelte:boundary>` 和 try/catch 有什么区别？**
A: `svelte:boundary` 捕获渲染时的同步/异步错误和 `$effect` 中的错误；try/catch 只能捕获同步错误。boundary 还可配合 `{#snippet pending}` 处理初次异步加载状态。

**Q: 什么时候用 `onerror`，什么时候用 `failed`？**
A: `failed` 是 UI——决定出错时**显示什么**；`onerror` 是副作用——决定出错时**做什么**（如上报到 Sentry）。两者完全独立，可以只用其一，也可以同时使用。

**Q: `svelte:element` 和普通组件有什么区别？**
A: `svelte:element` 渲染 HTML 标签（不是 Svelte 组件），适用于标签在运行时才确定的场景，如 CMS 内容。

**Q: 什么时候用 `namespace="svg"`？**
A: 当组件会作为 SVG 子元素被使用时（如 `<Icon.svelte>` 作为 `<svg>` 内 `<g>` 使用）。

**Q: `mount` / `hydrate` / `render` 怎么传 `transformError`？**
A: 都是同一签名——传一个 `(error) => JSON-stringifiable-object` 的函数：

```js
import { mount, hydrate } from 'svelte';
import { render } from 'svelte/server';

mount(App, { target, transformError: (e) => ({ message: e.message }) });
hydrate(App, { target, transformError: (e) => ({ message: 'Hydration failed' }) });
const { head, body } = await render(App, { transformError: (e) => ({ message: '...' }) });
```

**Q: 可以在多个组件中用同一个 `<svelte:head>` 块吗？**
A: 不行——`<svelte:head>` 是组件级的作用域。但你可以把 head 内容拆成 snippet，在多个组件中分别引入，或者封装一个 `<MetaTags>` 组件。

**Q: `<svelte:window>` 的事件处理器会影响性能吗？**
A: `onresize` / `onscroll` 这类高频事件建议做节流/防抖；其他事件开销可忽略。Svelte 会在组件销毁时自动移除监听器，无需手动清理。

---

## Examples

Practical examples demonstrating all `svelte:` special elements.

| File | Description |
|------|-------------|
| [examples/boundary-examples.md](./examples/boundary-examples.md) | Comprehensive examples for `<svelte:boundary>`: pending, failed, onerror, transformError, error reporting, nested boundaries |
| [examples/svelte-head-examples.md](./examples/svelte-head-examples.md) | `<svelte:head>` examples: SEO meta, Open Graph, structured data, dynamic stylesheets, SSR head extraction |
| [examples/svelte-element-examples.md](./examples/svelte-element-examples.md) | Examples for `<svelte:window>`, `<svelte:document>`, `<svelte:body>`, `<svelte:element>`, `<svelte:component>`, `<svelte:self>`, `<svelte:options>` |

### Quick Examples

- **svelte:boundary**: error fallback, loading skeleton, Sentry reporting, SSR `transformError`
- **svelte:window**: keyboard shortcuts, resize handling, scroll position binding
- **svelte:document**: page visibility, text selection tracking
- **svelte:body**: mouse enter/leave events
- **svelte:head**: dynamic `<title>`, Open Graph meta, JSON-LD, per-page stylesheets
- **svelte:element**: dynamic heading levels (h1-h6), conditional tags, SVG namespace
- **svelte:component**: dynamic component switching
- **svelte:options**: customElement, runes mode, SVG namespace

---

## References

Detailed reference documentation for all special elements and compiler options.

| File | Description |
|------|-------------|
| [references/boundary-reference.md](./references/boundary-reference.md) | Full reference for `<svelte:boundary>` — pending/failed/onerror, transformError (5.51+), nesting, what is/isn't caught |
| [references/svelte-head-reference.md](./references/svelte-head-reference.md) | Full reference for `<svelte:head>` — SSR behavior, supported elements, hydration, reactivity |
| [references/svelte-window-document-body-reference.md](./references/svelte-window-document-body-reference.md) | All events and bindable properties for `<svelte:window>` / `<svelte:document>` / `<svelte:body>` |
| [references/svelte-element-reference.md](./references/svelte-element-reference.md) | Reference for `<svelte:element>`, `<svelte:component>`, `<svelte:self>`, `<svelte:fragment>` |
| [references/svelte-options-reference.md](./references/svelte-options-reference.md) | Full reference for `<svelte:options>` — customElement, runes, namespace, css, and all compiler options |
