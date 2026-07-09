---
name: svelte
description: Svelte 5 UI 框架技能。当用户需要编写 Svelte 组件、使用 Runes 响应式系统（$state/$derived/$effect/$props）、处理模板语法（if/each/await/snippet）、配置样式与特殊元素、迁移 Svelte 4 代码、或在 SvelteKit/Vite 项目中开发时使用。
---

# Svelte Framework Reference (v5)

本技能覆盖 Svelte 5 框架。Svelte 5 的核心是 **Runes**（符文）系统，取代了 Svelte 4 的隐式响应式。编译器将 `.svelte` 组件直接转换为高度优化的 JavaScript，无虚拟 DOM。

## Capability Boundaries

### ✅ Strong Suits
1. 编写 Svelte 5 组件（`.svelte`），使用 Runes 响应式
2. `$state`、`$derived`、`$effect`、`$props`、`$bindable` 状态管理
3. 模板语法：`{#if}`、`{#each}`、`{#await}`、`{#snippet}`、`{@render}`
4. 作用域样式、`:global`、CSS 自定义属性
5. 特殊元素：`svelte:boundary`、`svelte:window`、`svelte:element`
6. Svelte 4 → Svelte 5 迁移（Legacy APIs 兼容）

### ⚠️ Requirements
1. 区分 Runes Mode（Svelte 5）和 Legacy Mode（Svelte 4）
2. SSR 和 CSR 行为差异需单独处理
3. SvelteKit 项目使用 SvelteKit 技能（本技能范围外）

### ❌ Out of Scope（替代方案）
1. SvelteKit 路由和 SSR 配置 → 使用 SvelteKit 技能
2. Vite/Webpack 构建配置 → 使用 vite/webpack 技能
3. 原生移动开发 → 使用对应移动技能

## When to use this skill

当用户需要编写、审查、调试 Svelte 组件，或讨论 `$state`/`$derived`/`$effect` 等 Runes、模板语法、组件通信、样式隔离、Svelte 4 迁移时使用本技能。

## Official Positioning

Svelte 是编译时响应的 UI 框架，无虚拟 DOM，直接操作真实 DOM。Svelte 5 引入 Runes 让响应式显式可控。

官方资源：[Svelte 文档](https://svelte.dev/docs/svelte) · [教程](https://svelte.dev/tutorial) · [GitHub](https://github.com/sveltejs/svelte) · [Discord](https://svelte.dev/chat)

## Critical: Runes System

| Rune | 用途 | Svelte 4 对应 |
|------|------|-------------|
| `$state` | 响应式状态 | `let` 声明 |
| `$derived` | 派生计算值 | `$: x = expr` |
| `$effect` | 副作用（DOM/库） | `$: { }` 块 |
| `$props` | 组件属性 | `export let` |
| `$bindable` | 可绑定 prop | `bind:` 指令 |
| `$inspect` | 开发调试 | `console.log` |

## Critical: $state / $derived / $effect

### $state — 响应式状态

```svelte
let count = $state(0);
let user = $state({ name: 'Ada', age: 30 });

// 深层代理：数组/对象属性变更自动触发更新
user.name = 'Bob';
items.push({ id: 1 });
```

**不使用深层代理：**
```js
let list = $state.raw([]); // 只能重新赋值，不能 .push()
```

**静态快照（传外部 API）：**
```svelte
console.log($state.snapshot(proxyObject));
```

### $derived — 派生值

```svelte
let count = $state(0);
let doubled = $derived(count * 2);

// 复杂派生用 $derived.by
let total = $derived.by(() => {
  return items.reduce((sum, item) => sum + item.price, 0);
});
```

**乐观 UI（Svelte 5.25+）**：`$derived` 值可直接覆盖用于即时反馈再回滚。

### $effect — 副作用

```svelte
$effect(() => {
  // 自动追踪 $state/$derived 依赖
  document.title = `count: ${count}`;
  return () => { /* 清理函数 */ };
});

// DOM 更新前运行
$effect.pre(() => { /* ... */ });
```

**不要用 `$effect` 同步状态** → 用 `$derived`。
**不要在 `$effect` 中直接修改 `$state`** → 会导致无限循环，改用 `$derived` 或函数绑定。

## Critical: $props / $bindable

### $props — 组件属性

```svelte
// 基础
let { name, age = 18, ...rest } = $props();

// 类型安全
let { title }: { title: string } = $props();

// 父组件使用
<MyComponent name="Alice" {age} />
```

### $bindable — 双向绑定 prop

```svelte
// 子组件 FancyInput.svelte
let { value = $bindable(), ...props } = $props();
<input bind:value />

// 父组件
let message = $state('hello');
<FancyInput bind:value={message} />
```

## Critical: Template Syntax

### 条件与列表

```svelte
{#if count > 10}
  <p>big</p>
{:else if count > 5}
  <p>medium</p>
{:else}
  <p>small</p>
{/if}

{#each items as item (item.id)}
  <li>{item.name}</li>
{:else}
  <p>空列表</p>
{/each}

{#await promise then value}
  <p>{value}</p>
{:catch e}
  <p>错误: {e}</p>
{/await}
```

### {#key ...} — 重建触发

```svelte
{#key count}
  <div transition:fade>{count}</div>
{/key}
```

### Snippets — 可复用标记块（取代 Slots）

```svelte
{#snippet card(item)}
  <div class="card">
    <h3>{item.title}</h3>
  </div>
{/snippet}

{@render card(item)}
```

**传递给组件：**
```svelte
// 隐式（推荐）
<Table {data}>
  {#snippet header()}
    <th>Name</th>
  {/snippet}
</Table>

// 显式
<Table {data} header={headerSnippet} />
```

**children 隐式片段：**
```svelte
// App: <Button>click</Button>
// Button:
let { children } = $props();
<button>{@render children()}</button>
```

### {@html ...} — 原始 HTML（注意 XSS）

```svelte
{@html rawContent}  <!-- 不转义，需信任来源 -->
```

### {@debug ...} — 调试标签

```svelte
{@debug user, count}        <!-- 变量变化时打印 -->
{@debug}                    <!-- 任意状态变化时断点 -->
```

## Critical: bind: Directive

```svelte
// 基础绑定
<input bind:value={msg} />
<textarea bind:value />
<input type="checkbox" bind:checked />
<input type="range" bind:value={n} min="0" max="100" />

// radio / checkbox 组
<input type="radio" bind:group={tortilla} value="Plain" />
<input type="checkbox" bind:group={fillings} value="Rice" />

// 文件
<input type="file" bind:files />

// 维度（readonly）
<div bind:clientWidth={w} bind:clientHeight={h} />

// 函数绑定（验证/转换）
<input bind:value={() => v, v => v = v.toLowerCase()} />
```

## Critical: use: Action

元素挂载时运行的函数（DOM 操作逻辑封装）：

```svelte
<script>
  function tooltip(node, content) {
    // node 是 DOM 元素
    const tip = createTip(node, content);
    return {
      destroy() { tip.destroy(); } // 清理
    };
  }
</script>
<div use:tooltip={'提示文字'}>hover</div>
```

## Critical: Transition & Animation

```svelte
<script>
  import { fade, fly } from 'svelte/transition';
  import { flip } from 'svelte/animate';
</script>

{#if visible}
  <div transition:fade>淡入淡出</div>
  <div transition:fly={{ y: 20, duration: 300 }}>飞入</div>
{/if}

{#each items as item (item.id)}
  <div animate:flip>{item.name}</div>
{/each}

// 全局过渡（跨控制流块）
<div transition:fade|global>
```

## Critical: Styling

```svelte
<style>
  /* 默认作用域化 */
  p { color: burlywood; }

  /* 全局 */
  :global(body) { margin: 0; }
  div :global(strong) { color: goldenrod; }

  /* CSS 自定义属性（传入） */
  .track { background: var(--track-color, #aaa); }
</style>
```

```svelte
<!-- 传入 CSS 自定义属性 -->
<Slider bind:value --track-color="black" />
```

## Critical: Special Elements

| 元素 | 用途 |
|------|------|
| `<svelte:boundary>` | 错误边界 + async pending UI（Svelte 5.3+） |
| `<svelte:window>` | 监听 window 事件和属性（`onkeydown`、`innerWidth` 等） |
| `<svelte:document>` | 监听 document 事件（`visibilitychange`） |
| `<svelte:body>` | 监听 body 事件（`mouseenter`） |
| `<svelte:element this={tag}>` | 动态渲染未知标签 |
| `<svelte:options>` | 编译器选项（`runes`、`namespace`、`customElement`） |

```svelte
<svelte:window onkeydown={handleKey} />
<svelte:element this={tag} xmlns="http://www.w3.org/2000/svg" />
<svelte:options customElement="my-element" />
```

## Critical: Lifecycle & Stores

### 生命周期

```svelte
<script>
  import { onMount, onDestroy, tick } from 'svelte';

  onMount(() => {
    console.log('mounted');
    return () => console.log('unmounted'); // 可选清理
  });
  onDestroy(() => { /* 销毁时 */ });
</script>
```

### tick — 等 DOM 更新完成

```svelte
$effect.pre(() => {
  tick().then(() => { /* DOM 已更新 */ });
});
```

### Stores（跨组件状态）

```svelte
import { writable, derived } from 'svelte/store';
const count = writable(0);
const doubled = derived(count, $c => $c * 2);

// 组件中用 $ 前缀访问
<script>
  import { count } from './store.js';
  console.log($count);
</script>
```

**Svelte 5 推荐**：用 `.svelte.js` 文件中的 `$state` 对象替代 store：
```js
// state.svelte.js
export const userState = $state({ name: '', count: 0 });
```

## Critical: Svelte 4 Migration

| Svelte 4 | Svelte 5 |
|---------|---------|
| `let x = 0`（顶层响应式） | `let x = $state(0)` |
| `$: x = a + b` | `let x = $derived(a + b)` |
| `$: { if (cond) x = v }` | `$effect(() => { if (cond) x = v })` |
| `export let prop` | `let { prop } = $props()` |
| `<slot />` | `{@render children()}` |
| `createEventDispatcher` | callback props 或 `$bindable` |

**自动迁移工具：**
```bash
npx svelte-migrate@latest svelte-5
```

### Legacy: Reactive $: Statements

```svelte
let a = 1, b = 2;
$: sum = a + b;           // 推导依赖
$: console.log(sum);
$: { total = 0; for (const x of list) total += x; }
```

### Legacy: export let

```svelte
export let name;
export let age = 18; // 默认值
```

### Legacy: Slots → Snippets

```svelte
// Svelte 4
<Component>
  <p slot="footer">Footer</p>
</Component>

// Svelte 5
<Component>
  {#snippet footer()}
    <p>Footer</p>
  {/snippet}
</Component>
```

### 数组更新注意

Svelte 4 基于赋值的响应式，`.push()` 不会触发更新：

```svelte
// ❌ 不触发
items.push(newItem);

// ✅ 触发
items.push(newItem);
items = items;
```

## Quick Fixes

| 问题 | 解决方案 |
|------|----------|
| 状态不更新 | 确认用了 `$state`（不是普通 `let`） |
| `$derived` 不生效 | 确保依赖在表达式中同步读取 |
| `$effect` 无限循环 | 不要在其中直接修改 `$state`，改用 `$derived` |
| prop 变异警告 | 不要修改 prop，用 `$bindable` 或 callback |
| 数组 `.push()` 不更新 | Svelte 4 需触发赋值；Svelte 5 深层代理自动更新 |
| 样式不生效 | 检查 scoped hash 是否覆盖，或用 `:global` |
| `bind:this` 为 undefined | 在 `onMount` 之后使用 |
| Svelte 4 代码迁移 | `npx svelte-migrate@latest svelte-5` |

## Audience

| 用户类型 | 使用方式 |
|----------|----------|
| **Svelte 5 新手** | 从 Runes（$state/$derived/$effect）开始 |
| **Svelte 4 开发者** | 重点看 Legacy APIs 和迁移指南 |
| **React/Vue 开发者** | 对比理解：Svelte 无虚拟 DOM、编译时响应式推导 |
| **SSR 项目** | 使用 SvelteKit；纯 Svelte 用 `render()`/`hydrate()` |

## Gotchas

1. **`$state` 是深层代理** — 解构后丢失响应式
2. **`$effect` 不应同步状态** — 用 `$derived` 做派生；`$effect` 仅用于副作用
3. **Runes 仅限 `.svelte/.svelte.js/.svelte.ts`** — 普通 `.js` 文件不可用
4. **Snippet 取代 Slot** — Svelte 4 的 `<slot>` 在 Svelte 5 中已废弃
5. **事件委托** — 大多数事件自动委托到根节点，无需手动 `stopPropagation`
6. **SSR 保护** — 浏览器 API（`window`/`document`）在 SSR 时需用 `typeof window !== 'undefined'` 保护
7. **Scoped 样式优先级** — 哈希类增加 0-1-0 特异性，可能覆盖全局样式
8. **`{@html}` XSS 风险** — 不转义 HTML，确保内容可信

## FAQ

**Q: 什么时候用 `$state` vs `$derived`？**
A: `$state` 用于可变状态；`$derived` 用于从已有状态计算得出的只读值。任何"X 变则 Y 自动更新"用 `$derived`。

**Q: `bind:` 和普通 props 的区别？**
A: Props 单向流动（父→子）；`bind:` 允许子组件直接修改父组件状态。谨慎使用，过度使用使数据流难以追踪。

**Q: Svelte 和 React/Vue 的核心区别？**
A: Svelte 无虚拟 DOM，编译时生成精确的 DOM 更新代码，运行时开销极低。响应式基于编译器推导的依赖追踪。

**Q: 如何做 SSR？**
A: 使用 SvelteKit。纯 Svelte 也支持 `render()`（服务端）和 `mount()`（客户端）。

**Q: 支持 TypeScript 吗？**
A: 原生支持，在 `<script lang="ts">` 中编写，无需额外配置。
