---
name: svelte-special-elements
description: Svelte 5 特殊元素技能。当用户需要使用 svelte:boundary、svelte:window、svelte:element、svelte:options 等特殊元素，或配置编译器选项、自定义元素时使用。
---

# Svelte Special Elements Reference (Svelte 5)

本技能覆盖 Svelte 模板中的特殊元素（以 `svelte:` 为前缀），用于处理边界、窗口事件、动态标签、编译器选项等场景。

## When to use this skill

当用户需要捕获渲染错误/异步 pending、使用 `svelte:window` 事件、动态渲染元素、配置编译器选项、或创建自定义元素时使用本技能。

## Critical: `<svelte:boundary>`

错误和异步边界（Svelte 5.3+），"隔离"应用局部以处理错误和 pending 状态：

```svelte
<svelte:boundary onerror={handleError}>
  <RiskyComponent />

  {#snippet failed(error)}
    <p>出错了: {error.message}</p>
  {/snippet}

  {#snippet pending()}
    <p>加载中...</p>
  {/snippet}
</svelte:boundary>
```

**注意**：渲染过程中的错误和异步操作会被捕获；事件处理器、`setTimeout` 中的错误不会被捕获。

## Critical: `<svelte:window>`

监听 window 事件和绑定 window 属性：

```svelte
<svelte:window onkeydown={handleKey} onresize={handleResize} />
<svelte:window bind:scrollX bind:scrollY bind:innerWidth bind:innerHeight />
```

**可绑定属性**：`innerWidth`、`innerHeight`、`outerWidth`、`outerHeight`、`scrollX`、`scrollY`、`online`。

**可监听事件**：`keydown`、`keyup`、`resize`、`scroll`、`beforeinput`、`click`、`touchstart` 等所有 window 级别事件。

> `<svelte:window>` 只能出现在组件顶层，不能在块级元素或条件块内。

## Critical: `<svelte:document>`

监听 document 级别事件（`window` 不支持的事件）：

```svelte
<svelte:document onvisibilitychange={handleVisibility} />
<svelte:document {@attach myAttachment} />
```

**可绑定属性**（readonly）：`activeElement`、`fullscreenElement`、`pointerLockElement`、`visibilityState`。

## Critical: `<svelte:body>`

监听 body 元素事件（如 `mouseenter`、`mouseleave`，这些事件不在 window 上触发）：

```svelte
<svelte:body onmouseenter={handleMouseenter} onmouseleave={handleMouseleave} />
```

同样支持 `use:` action。

## Critical: `<svelte:head>`

向 `document.head` 插入内容（SSR 时单独暴露）：

```svelte
<svelte:head>
  <title>Hello world!</title>
  <meta name="description" content="SEO description" />
</svelte:head>
```

## Critical: `<svelte:element>`

动态渲染未知标签（如来自 CMS 或数据库）：

```svelte
<script>
  let tag = $state('hr');
</script>

<svelte:element this={tag}>内容</svelte:element>
<svelte:element this={tag} xmlns="http://www.w3.org/2000/svg" />
```

- `this` 为 nullish → 元素不渲染
- `this` 为 void 元素（`br`）但有子元素 → 运行时错误
- Svelte 自动推断 namespace（svg、mathml），可用 `xmlns` 显式指定

**唯一支持的绑定**：`bind:this`（因为通用元素不支持 Svelte 内建绑定）

## Critical: `<svelte:options>`

设置编译器选项：

```svelte
<svelte:options runes={true} />
<svelte:options namespace="svg" />
<svelte:options customElement="my-element" />
<svelte:options css="injected" />
```

| 选项 | 说明 |
|------|------|
| `runes={true/false}` | 强制进入/退出 Runes Mode |
| `namespace="html/svg/mathml"` | 组件命名空间 |
| `customElement="tag-name"` | 编译为自定义元素 |
| `css="injected"` | 样式内联注入（SSR → style 标签；CSR → JS） |

## Quick Fixes

| 问题 | 解决方案 |
|------|----------|
| 想监听 visibilitychange | 用 `<svelte:document>` 而非 `<svelte:window>` |
| 想监听 mouseenter/mouseleave | 用 `<svelte:body>` |
| 动态标签不渲染 | 检查 `this` 是否为 nullish |
| 样式不生效 | 用 `css="injected"` 强制内联 |

## Gotchas

1. **只能出现在顶层** — `svelte:window/document/body/head/element` 不能在 `{#if}` 或其他块级元素内
2. **SSR 行为不同** — `<svelte:head>` 在 SSR 时内容单独暴露，不在 body 内
3. **`svelte:element` 是通用绑定** — 不支持 `bind:value` 等元素特有绑定

## FAQ

**Q: `<svelte:boundary>` 和 try/catch 有什么区别？**
A: `svelte:boundary` 捕获渲染时的同步/异步错误；try/catch 只能捕获同步错误。boundary 还可配合 `{:snippet pending}` 处理异步加载状态。

**Q: `svelte:element` 和普通组件有什么区别？**
A: `svelte:element` 渲染 HTML 标签（不是 Svelte 组件），适用于标签在运行时才确定的场景，如 CMS 内容。

**Q: 什么时候用 `namespace="svg"`？**
A: 当组件会作为 SVG 子元素被使用时（如 `<Icon.svelte>` 作为 `<svg>` 内 `<g>` 使用）。
