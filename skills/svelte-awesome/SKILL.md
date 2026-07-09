---
name: svelte-awesome
description: Svelte 5 入门与导航技能。当用户需要了解 Svelte 5 全貌、选择合适的学习路径、或需要本技能的 8 个子技能协同工作时加载此入口技能。
---

# Svelte 5 — 全技能导航入口

本技能是 Svelte 5 技能的**编排入口**，提供框架全貌、学习路径指引和子技能协同指南。

## 关于 Svelte 5

Svelte 5 是新一代 UI 框架，核心特性：

- **无虚拟 DOM** — 编译器直接生成精确的 DOM 操作代码，运行时零开销
- **Runes 响应式** — `$state`/`$derived`/`$effect` 等显式符文替代隐式响应式
- **原生 TypeScript** — 无需额外配置
- **SvelteKit** — 官方应用框架，支持 SSR、静态站点、API 路由

## 技能架构图

```
svelte-awesome（入口 · 本技能）
├── svelte-runes          ← Runes 响应式系统
├── svelte-template-syntax ← 模板语法
├── svelte-styling       ← 样式与 CSS
├── svelte-special-elements ← 特殊元素
├── svelte-runtime       ← 运行时 API
├── svelte-misc          ← TypeScript / 迁移 / FAQ
├── svelte-reference     ← API 参考
└── svelte-legacy-apis  ← Svelte 4 兼容
```

## 子技能速查表

| 技能 | 何时加载 | 核心内容 |
|------|---------|---------|
| **svelte-runes** | 使用 `$state`/`$derived`/`$effect`/`$props` 时 | 响应式系统、状态传递、副作用 |
| **svelte-template-syntax** | 编写组件模板时 | if/each/await/snippet/事件绑定 |
| **svelte-styling** | 处理 CSS 样式时 | Scoped styles、:global、CSS 变量 |
| **svelte-special-elements** | 使用 `svelte:boundary`/`window`/`element` 时 | 特殊元素、编译器选项 |
| **svelte-runtime** | 跨组件状态、生命周期、测试时 | Stores、Context、mount/render、Vitest |
| **svelte-misc** | TypeScript、迁移、浏览器兼容性时 | TS 标注、Svelte 4→5 迁移、FAQ |
| **svelte-reference** | 查阅具体 API 签名时 | store/action/transition/easing、错误代码 |
| **svelte-legacy-apis** | 维护 Svelte 4 代码时 | `$:`、`export let`、Slots、EventDispatcher |

## 学习路径

### 路径一：Svelte 新手

```
svelte-awesome → svelte-runes → svelte-template-syntax → svelte-styling → svelte-runtime
```

### 路径二：从 Vue/React 迁移

```
svelte-awesome → svelte-runes → svelte-template-syntax → svelte-misc（迁移章节）
```

### 路径三：从 Svelte 4 迁移

```
svelte-awesome → svelte-misc（Svelte 4 Migration）→ svelte-legacy-apis（对照参考）
```

## 典型任务与技能匹配

| 任务 | 推荐技能 |
|------|---------|
| 写一个计数器组件 | `svelte-runes` |
| 渲染列表、条件分支 | `svelte-template-syntax` |
| 用 snippet 复用标记块 | `svelte-template-syntax`（snippet 章节）|
| 组件样式隔离、CSS 变量传递 | `svelte-styling` |
| 错误边界、异步 pending UI | `svelte-special-elements` |
| 跨组件共享状态 | `svelte-runtime`（Context/Stores 章节）|
| 单元测试、Vitest 配置 | `svelte-runtime`（Testing 章节）|
| TypeScript 类型标注 | `svelte-misc`（TypeScript 章节）|
| Svelte 4 项目迁移到 Svelte 5 | `svelte-misc` |
| 查阅 `transition:fade` API | `svelte-reference` |
| 理解 `bind:value` 双向绑定 | `svelte-template-syntax`（bind 章节）|
| `bind:this` 获取组件实例 | `svelte-runtime`（mount 章节）|
| 维护 Legacy Svelte 4 代码 | `svelte-legacy-apis` |

## Svelte 5 核心概念速记

### Runes 响应式
```svelte
let count = $state(0);           // 响应式状态
let doubled = $derived(count * 2); // 派生值
$effect(() => { document.title = count; }); // 副作用
let { name } = $props();         // 属性
let { value = $bindable() } = $props(); // 可绑定属性
```

### 模板
```svelte
{#if count > 0}
  <p>counting...</p>
{/if}

{#each items as item (item.id)}
  <li>{item.name}</li>
{/each}

{#snippet card(item)}
  <div>{item.title}</div>
{/snippet}
{@render card(item)}
```

### 事件
```svelte
<!-- Svelte 5: onclick 不是 on:click -->
<button onclick={() => count++}>+</button>
```

## 官方资源

| 资源 | 链接 |
|------|------|
| Svelte 文档 | https://svelte.dev/docs/svelte |
| 官方教程 | https://svelte.dev/tutorial |
| Playground | https://svelte.dev/playground |
| SvelteKit | https://kit.svelte.dev |
| GitHub | https://github.com/sveltejs/svelte |
| Discord | https://svelte.dev/chat |

## Gotchas

1. **Svelte 5 进入 Runes Mode** — 组件中使用了任意一个 Rune（`$state` 等）即进入 Runes Mode，Legacy 语法不再可用
2. **无虚拟 DOM** — Svelte 编译时生成精确 DOM 操作，不存在 diffing 和 patch
3. **`$state` 是深层代理** — 解构会丢失响应式；大数组用 `$state.raw` 避免代理开销
4. **`$effect` 是副作用专用** — 不要用它做派生计算，那是 `$derived` 的职责
5. **Snippet 取代 Slot** — 新代码用 `{#snippet}` + `{@render}`，不再使用 `<slot>`

## FAQ

**Q: Svelte 5 和 Svelte 4 可以共存吗？**
A: 可以。Svelte 5 仍支持 Legacy Mode（Svelte 4 语法），可渐进迁移。

**Q: Svelte 和 React/Vue 的核心区别？**
A: Svelte 无虚拟 DOM，编译时生成精确 DOM 操作代码，运行时零框架开销。响应式基于编译器推导的依赖追踪，无需声明依赖。

**Q: 需要先学 Svelte 4 吗？**
A: 不需要。直接学 Svelte 5 Runes 系统即可，Svelte 4 的 Legacy 知识仅在维护旧项目时有帮助。

**Q: Svelte 适合什么场景？**
A: 适合所有 Web UI 场景：从简单组件到复杂应用。配合 SvelteKit 支持 SSR、静态站点、API 路由等。
