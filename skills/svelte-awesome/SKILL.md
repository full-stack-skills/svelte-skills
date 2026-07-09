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
├── Svelte 核心 5.x ──────────
│   ├── svelte-runes           ← Runes 响应式系统
│   ├── svelte-template-syntax ← 模板语法
│   ├── svelte-styling         ← 样式与 CSS
│   ├── svelte-special-elements← 特殊元素
│   ├── svelte-runtime         ← 运行时 API（Stores / Context / mount）
│   ├── svelte-lifecycle       ← 生命周期 / $effect / 副作用
│   ├── svelte-misc            ← TypeScript / 自定义元素 / 迁移 / FAQ
│   ├── svelte-reference       ← API 参考
│   └── svelte-legacy-apis     ← Svelte 4 兼容
├── SvelteKit 配套 ───────────
│   ├── sveltekit-overview     ← SvelteKit 全貌 / 路由 / 加载
│   ├── sveltekit-data         ← 数据加载 / form / actions
│   ├── sveltekit-config       ← 配置 / adapter / 钩子
│   └── sveltekit-advanced     ← 高级（hooks / service worker / hooks 进阶）
└── 工具链 ───────────────────
    ├── svelte-cli             ← sv / create-svelte / npx sv
    └── svelte-ai              ← Svelte AI 助手 / MCP / 编辑器集成
```

## 子技能速查表

| 技能 | 何时加载 | 核心内容 |
|------|---------|---------|
| **svelte-runes** | 使用 `$state`/`$derived`/`$effect`/`$props` 时 | 响应式系统、状态传递、副作用 |
| **svelte-template-syntax** | 编写组件模板时 | if/each/await/snippet/事件绑定 |
| **svelte-styling** | 处理 CSS 样式时 | Scoped styles、:global、CSS 变量 |
| **svelte-special-elements** | 使用 `svelte:boundary`/`window`/`element` 时 | 特殊元素、编译器选项 |
| **svelte-runtime** | 跨组件状态、生命周期、测试时 | Stores、Context、mount/render、Vitest |
| **svelte-misc** | TypeScript、迁移、浏览器兼容性时 | TS 标注、自定义元素、Svelte 4→5 迁移、FAQ |
| **svelte-reference** | 查阅具体 API 签名时 | store/action/transition/easing、错误代码 |
| **svelte-lifecycle** | 处理组件挂载/卸载/副作用时 | `$effect` / `$effect.pre` / `onMount` / `onDestroy` / `$inspect` |
| **svelte-legacy-apis** | 维护 Svelte 4 代码时 | `$:`、`export let`、Slots、EventDispatcher |

## SvelteKit 配套技能

当目标是**应用/路由/SSR/数据加载**时，从 SvelteKit 技能开始：

| 技能 | 何时加载 | 核心内容 |
|------|---------|---------|
| **sveltekit-overview** | 创建项目、项目类型、Web 标准、Routing | 项目结构、`+page.svelte` / `+page.server.js` / `+layout`、路由分组、错误边界、SSR/SSG/SPA/MPA |
| **sveltekit-data** | 数据加载、表单、页面选项 | `+page.js` / `+page.server.js` 的 `load`、form actions、`use:enhance`、prerender/ssr/csr |
| **sveltekit-advanced** | 状态管理、Remote functions、Hooks、Errors、Service workers、$app/* | SSR 安全 stores、`query`/`command`、`handle`/`sequence`、`+error.svelte`、`$app/forms`/`navigation`/`state`、`$lib` |
| **sveltekit-config** | 构建、Adapters、Auth、Performance、Images、Migration | `adapter-auto`/`node`/`static`/`cloudflare`/`netlify`/`vercel`、`@sveltejs/enhanced-img`、SvelteKit v1/Sapper 迁移 |

### 路径四：从 Svelte 组件到 SvelteKit 应用

```
svelte-awesome
   ↓
sveltekit-overview（项目骨架）
   ↓
sveltekit-data（数据流）
   ↓
sveltekit-config（adapter / env）
   ↓
sveltekit-advanced（按需）
```

## 工具集成

| 技能 | 何时加载 | 核心内容 |
|------|---------|---------|
| **svelte-cli** | sv 命令（项目创建、添加集成、迁移脚本、类型检查） | `sv create` / `sv add`（drizzle/tailwind/prettier/eslint/playwright/storybook/vitest/mdsvex/paraglide/better-auth/mcp 等）、`sv check`、`sv migrate svelte-5`/`sveltekit-2`/... |
| **svelte-ai** | MCP server、AI 集成 | Svelte MCP server（svelte-autofixer / list-sections / get-documentation）、Claude Code / Cursor / Copilot / Codex / OpenCode / VS Code 配置、`svelte-task` 提示、`sv` CLI 的 svelte-ai 子命令、`svelte-code-writer` 子智能体 |

## 进阶主题

| 技能 | 何时加载 | 核心内容 |
|------|---------|---------|
| **svelte-lifecycle** | 生命周期 / Stores / Context / Testing | `onMount` / `onDestroy` / `tick` 钩子、`writable` / `readable` / `derived` stores、`createContext` / `setContext` / `getContext`、Vitest / Storybook / Playwright |

> 与 `svelte-runtime`（Stores / Context / mount）互补：**lifecycle** 把生命周期 + Stores + Context + 测试聚合在一起，`runtime` 偏向 runtime API（mount / render）的细节。

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

### 路径四：构建 SvelteKit 应用

```
svelte-awesome
  → svelte-cli（创建项目）
  → svelte-runes（响应式基础）
  → svelte-template-syntax
  → sveltekit-overview（项目骨架）
  → sveltekit-data（数据流 / form actions）
  → sveltekit-config（adapter / env）
  → svelte-lifecycle（副作用与数据同步）
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
| 把组件编译为 Web Component | `svelte-misc`（Custom Elements 章节）|
| 控制副作用时机（DOM 更新前后、清理） | `svelte-lifecycle` |
| 调试响应式不更新 | `svelte-lifecycle`（`$inspect` / `$effect.tracking`）|
| 初始化 SvelteKit 项目 | `svelte-cli` |
| 给 SvelteKit 加 adapter/tailwind/drizzle | `svelte-cli` |
| 配置 AI 编辑器（Cursor/Windsurf）的 Svelte 上下文 | `svelte-ai` |
| SvelteKit 路由 / `+page.svelte` / `+layout.svelte` | `sveltekit-overview` |
| SvelteKit `load` 函数 / form actions | `sveltekit-data` |
| 配置 SvelteKit adapter / aliases / env | `sveltekit-config` |
| SvelteKit `handle` 钩子 / service worker | `sveltekit-advanced` |
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

## Cross-References

其它技能的协作指引：

- 副作用 / 生命周期 / `$effect` 清理 → `svelte-lifecycle`
- SvelteKit 路由 / 数据 / 配置 → `sveltekit-overview` → `sveltekit-data` / `sveltekit-config` / `sveltekit-advanced`
- 项目脚手架 (`npx sv create`) → `svelte-cli`
- AI 编辑器 / LLM 辅助开发（Cursor / Windsurf / Zed 等）→ `svelte-ai`

## Gotchas

1. **Svelte 5 进入 Runes Mode** — 组件中使用了任意一个 Rune（`$state` 等）即进入 Runes Mode，Legacy 语法不再可用
2. **无虚拟 DOM** — Svelte 编译时生成精确 DOM 操作，不存在 diffing 和 patch
3. **`$state` 是深层代理** — 解构会丢失响应式；大数组用 `$state.raw` 避免代理开销
4. **`$effect` 是副作用专用** — 不要用它做派生计算，那是 `$derived` 的职责；副作用清理与调试见 `svelte-lifecycle`
5. **Snippet 取代 Slot** — 新代码用 `{#snippet}` + `{@render}`，不再使用 `<slot>`
6. **SvelteKit ≠ Svelte** — 仅做组件化/静态站点 → 用 Svelte；要做路由/SSR/表单服务端处理 → 进入 `sveltekit-overview`

## FAQ

**Q: Svelte 5 和 Svelte 4 可以共存吗？**
A: 可以。Svelte 5 仍支持 Legacy Mode（Svelte 4 语法），可渐进迁移。

**Q: Svelte 和 React/Vue 的核心区别？**
A: Svelte 无虚拟 DOM，编译时生成精确 DOM 操作代码，运行时零框架开销。响应式基于编译器推导的依赖追踪，无需声明依赖。

**Q: 需要先学 Svelte 4 吗？**
A: 不需要。直接学 Svelte 5 Runes 系统即可，Svelte 4 的 Legacy 知识仅在维护旧项目时有帮助。

**Q: Svelte 适合什么场景？**
A: 适合所有 Web UI 场景：从简单组件到复杂应用。配合 SvelteKit 支持 SSR、静态站点、API 路由等。

## Examples

离线可执行示例，配合本技能使用：

- [getting-started.md](examples/getting-started.md) — 项目创建、首个组件、SvelteKit 路由、API 路由、环境变量
- [component-patterns.md](examples/component-patterns.md) — Props、Snippet、事件、绑定、响应式、跨组件状态完整示例
- [sveltekit-bootstrap.md](examples/sveltekit-bootstrap.md) — 用 `npx sv create` 创建 SvelteKit 项目到首个路由
- [lifecycle-patterns.md](examples/lifecycle-patterns.md) — `$effect` 清理、`onMount`、`onDestroy`、`$inspect`

### 子技能示例

- 路由 / 数据加载 / `+page.svelte` / `+layout.svelte` → `../sveltekit-overview/examples/`
- `load` 函数 / form actions / `use:enhance` → `../sveltekit-data/examples/`
- adapter / env / `$lib` / hooks / service worker → `../sveltekit-advanced/examples/`
- `svelte.config.js` / 构建 / 部署 / enhanced-img → `../sveltekit-config/examples/`
- `sv create` / `sv add` / `sv migrate` / `sv check` → `../svelte-cli/examples/`
- Svelte MCP server / Cursor rules → `../svelte-ai/examples/`
- 生命周期 / Stores / Context / Vitest / Storybook → `../svelte-lifecycle/examples/`

## References

深入参考文档：

- [runes-overview.md](references/runes-overview.md) — Runes 原理、代理行为、依赖追踪规则、模块级 $state
- [template-syntax.md](references/template-syntax.md) — 模板语法速查、{#each}/{#await}/{#snippet} 完整用法、bind: 指令对照
- [styling.md](references/styling.md) — Scoped Styles/:global/Custom Properties/Tailwind 集成
- [migration.md](references/migration.md) — Svelte 4→5 迁移对照、自动迁移脚本、Legacy Mode
- [skill-map.md](references/skill-map.md) — 全部 16 个 svelte-* / sveltekit-* 技能的全景图与加载时机
- [routing-data.md](references/routing-data.md) — SvelteKit 路由 + 数据加载速查

### 子技能参考

- SvelteKit 项目结构 / 路由约定 / Web 标准 → `../sveltekit-overview/references/`
- 数据加载 / form actions / 页面选项 / `use:enhance` → `../sveltekit-data/references/`
- SSR 安全 stores / Remote functions / Hooks / `$app/*` → `../sveltekit-advanced/references/`
- Adapters / 构建 / 性能 / Images / Migration → `../sveltekit-config/references/`
- `sv` 命令完整参数 / 自定义 add-on → `../svelte-cli/references/`
- MCP tools / AI 集成清单 → `../svelte-ai/references/`
- Lifecycle / Stores / Context / Testing 深入 → `../svelte-lifecycle/references/`
