---
name: svelte-misc
description: Svelte 5 杂项技能。当用户需要了解 TypeScript 支持、自定义元素、浏览器兼容性、Svelte 4→5 迁移、FAQ 时使用。
---

# Svelte Misc Reference (Svelte 5)

本技能覆盖 Svelte 5 的 TypeScript 支持、自定义元素、浏览器兼容性、Svelte 4→5 迁移指南和常见问答。

## When to use this skill

当用户需要 TypeScript 类型标注、创建自定义 Web 组件、查看浏览器兼容性、迁移 Svelte 4 项目到 Svelte 5、或了解 Svelte 与其他框架的区别时使用本技能。

## Critical: TypeScript

Svelte 原生支持 TypeScript，无需额外配置（Vite 项目）。

### 基本使用

```svelte
<script lang="ts">
  let { name }: { name: string } = $props();
  let count = $state(0);
  const double = $derived(count * 2);
</script>
```

### 类型标注 Props

```svelte
<script lang="ts">
  interface Props {
    title: string;
    count?: number;
    onClick?: () => void;
    children?: import('svelte').Snippet;
  }
  let { title, count = 0, onClick, children }: Props = $props();
</script>
```

### 导入类型

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';
  let { children }: { children: Snippet } = $props();
</script>
```

### 泛型组件

```svelte
<script lang="ts" generics="T extends string">
  let { items }: { items: T[] } = $props();
</script>
```

### 类型工具

```ts
import type { Component, ComponentProps } from 'svelte';
type MyProps = ComponentProps<MyComponent>;
```

## Critical: Custom Elements

Svelte 可编译为 Web Components（自定义元素）：

### 基础用法

```svelte
<svelte:options customElement="my-element" />

<script>
  let { count = 0 } = $props();
</script>

<button onclick={() => count++}>{count} clicks</button>
```

### 生命周期

```svelte
<svelte:options
  customElement={{
    tag: 'my-counter',
    props: { count: { type: Number } },
    observedAttributes: ['count']
  }}
/>
```

## Critical: Browser Support

| 环境 | 支持情况 |
|------|---------|
| 现代浏览器 | ✅ 完全支持 |
| IE 11 | ❌ 不支持 |
| SSR | ✅ 支持（无 DOM 依赖部分） |

### SSR 浏览器 API 保护

```svelte
<script>
  if (typeof window !== 'undefined') {
    // 浏览器专用代码
    window.addEventListener('resize', handleResize);
  }
</script>
```

> 建议用 `<svelte:window>` 替代直接使用 `window.addEventListener`

## Critical: Svelte 4 Migration Guide

### 最小版本要求

- Node 16+
- SvelteKit 1.20.4+（如使用）
- `vite-plugin-svelte` 2.4.1+
- TypeScript 5+

### 核心迁移对照

| Svelte 4 | Svelte 5 |
|---------|---------|
| `let x = 0`（顶层响应式） | `let x = $state(0)` |
| `$: x = a + b` | `let x = $derived(a + b)` |
| `$: { if (cond) x = v }` | `$effect(() => { if (cond) x = v })` |
| `export let prop` | `let { prop } = $props()` |
| `<slot />` | `{@render children()}` |
| `createEventDispatcher` | callback props |
| `on:click={fn}` | `onclick={fn}` |
| `bind:this={ref}` | `bind:this={ref}`（相同） |

### 自动迁移工具

```bash
npx svelte-migrate@latest svelte-5 my-project
```

## Critical: Svelte 5 Migration Guide

### 主要变化

| 变化 | 说明 |
|------|------|
| Runes 系统 | `$state`/`$derived`/`$effect` 替代隐式响应式 |
| Snippets | `{#snippet}` + `{@render}` 替代 Slots |
| 事件属性 | `onclick={fn}` 替代 `on:click={fn}` |
| `$effect` 行为 | 必须显式追踪依赖 |
| `$state` 深层代理 | 对象/数组自动 Proxy |

### 移除的 Legacy 功能

| 功能 | 替代 |
|------|------|
| `let` 顶层响应式 | `$state` |
| `$:` 语句 | `$derived`/`$effect` |
| `export let` | `$props` |
| `createEventDispatcher` | callback props / `$bindable` |
| Slots | Snippets |
| `on:event` | `onevent` |
| `$$props`/`$$restProps` | 解构 `$props()` |
| `beforeUpdate`/`afterUpdate` | `$effect.pre` / `$effect` |

### 新增功能（了解）

- `$state.eager`：立即 UI 更新
- `$effect.pre`：DOM 更新前 effect
- `$effect.tracking`：追踪上下文检测
- `$state.snapshot`：静态快照
- `hydratable`：SSR 数据复用
- `fork()`：异步预加载
- `{#key}` 过渡触发
- `<svelte:boundary>`：错误/pending 边界
- `createContext`：类型安全 Context

## FAQ

**Q: Svelte 和 Vue/React 的核心区别？**
A: Svelte 无虚拟 DOM——编译时生成精确的 DOM 更新代码，运行时零框架开销。响应式基于编译器推导的依赖追踪，无需声明依赖。

**Q: Svelte 5 和 Svelte 4 可以共存吗？**
A: 可以。Svelte 5 仍然支持 Legacy Mode，Svelte 4 的 `$:` 和 `export let` 语法可以正常工作。可以在同一项目渐进迁移。

**Q: 如何升级到 Svelte 5？**
A: ① 升级 `svelte` 和 `sveltekit`/`vite-plugin-svelte` 包；② 运行 `npx svelte-migrate@latest svelte-5` 自动迁移；③ 逐步将组件迁移到 Runes Mode。

**Q: Svelte 支持 SSR 吗？**
A: 原生支持。使用 `render()` 函数（服务端）或 SvelteKit（官方推荐）。

**Q: Svelte 支持 TypeScript 吗？**
A: 原生支持。在 `<script lang="ts">` 中直接编写 TypeScript。

**Q: Svelte 有虚拟 DOM 吗？**
A: 没有。Svelte 编译器直接生成精确的 DOM 操作代码（`element.textContent = ...`、`element.style.color = ...`），而非通过虚拟 DOM diffing。

**Q: 哪些浏览器支持 Svelte 5？**
A: 所有现代浏览器（Chrome 80+、Firefox 75+、Safari 14+、Edge 80+）。不支持 IE。

## Quick Fixes

| 问题 | 解决方案 |
|------|----------|
| Svelte 4 代码报错 | 确认组件未使用 `$state`（进入 Runes Mode） |
| `$:` 不生效 | 升级到 Svelte 5，Legacy Mode 仍支持 |
| TypeScript 报错 | 确认 `<script lang="ts">` 和 TypeScript 5+ |
| 迁移后打包变大 | 检查是否正确配置了 tree-shaking |

## Gotchas

1. **Svelte 5 自动进入 Runes Mode** — 只要使用了任何 Rune（`$state` 等），组件即为 Runes Mode
2. **Legacy Mode 可通过 `<svelte:options runes={false}>` 强制启用**
3. **Svelte 4 的 `let` 顶层响应式在 Svelte 5 Legacy Mode 仍有效**
4. **事件属性 `on:click` 已废弃** — 但在 Legacy Mode 仍可用（会警告）
