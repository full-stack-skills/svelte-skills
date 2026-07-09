---
name: svelte-legacy-apis
description: Svelte 5 Legacy API 技能。当用户需要维护或迁移 Svelte 4 代码，理解 Reactive let/$:/export let/Slots/createEventDispatcher/$$props 等已废弃但仍可用的 Legacy API 时使用。
---

# Svelte Legacy APIs Reference (Svelte 4 → Svelte 5)

本技能覆盖 Svelte 4 的 Legacy API，这些 API 在 Svelte 5 中仍可用（Legacy Mode），但推荐迁移到 Runes。本技能仅用于理解和迁移 Legacy 代码，**新代码应使用 Runes**。

## When to use this skill

当用户需要维护 Svelte 4 项目、理解 `$:` reactive 语句、`export let` props、`createEventDispatcher`、Slots、或需要将 Legacy 代码迁移到 Svelte 5 Runes Mode 时使用本技能。

## Critical: Reactive let declarations

在 Legacy Mode（未使用 Runes 的组件），顶层 `let` 声明自动成为响应式：

```svelte
<script>
  let count = 0; // 自动响应式
</script>

<button on:click={() => count++}>
  {count}
</button>
```

> Svelte 5 的 Runes Mode：`let count = $state(0)`。

### 数组/对象变更注意

Legacy 响应式基于**赋值**，数组方法（`.push()`、`.splice()`）不触发更新：

```svelte
<script>
  let items = [1, 2, 3];

  function addItem() {
    items.push(4);    // ❌ 不触发更新
    items = items;    // ✅ 触发
    // 或
    items = [...items, 4]; // ✅ 触发
  }
</script>
```

## Critical: Reactive $: statements

在 Legacy Mode，`$:` 标签语句创建响应式语句：

```svelte
<script>
  let a = 1;
  let b = 2;

  $: sum = a + b;              // 依赖 a, b
  $: console.log(`sum = ${sum}`); // 依赖 sum
</script>
```

### 拓扑排序

`$:` 语句按依赖拓扑排序（顺序无关）：

```svelte
$: z = y;    // z 依赖 y
$: y = x;    // y 依赖 x
$: x = 5;    // x 无依赖
```

### 块级语句

```svelte
$: {
  total = 0;
  for (const item of items) total += item.value;
}
```

## Critical: export let

Legacy Mode 中用 `export let` 声明 props：

```svelte
<script>
  export let name;              // 必需 prop
  export let age = 18;          // 带默认值
  export let items = [];         // 数组默认值
</script>
```

> Svelte 5 Runes：`let { name, age = 18 } = $props()`

### 重新导出（API）

```svelte
<script>
  export function greet(name) {
    alert(`Hello ${name}!`);
  }
  export const VERSION = '1.0';
</script>
```

父组件通过 `bind:this` 调用：

```svelte
<script>
  let greeter;
</script>
<Greeter bind:this={greeter} />
<button on:click={() => greeter.greet('world')}>greet</button>
```

### prop 重命名

```svelte
<script>
  export { foo as bar }; // 对外叫 bar，内部叫 foo
</script>
```

## Critical: $$props / $$restProps

### $$props（获取所有传入 props）

```svelte
<script>
  let { foo, ...rest } = $$props;
  // foo 是显式 prop，rest 是其余所有 props
</script>
```

> Svelte 5 Runes：`let { foo, ...rest } = $props()`

### $$restProps（仅剩余 props）

```svelte
<script>
  export let required;
  let { ...rest } = $$props;
</script>
<!-- required 不在 rest 中 -->
```

## Critical: createEventDispatcher

Legacy 的事件分发机制（已被 Svelte 5 callback props 取代）：

```svelte
<script>
  import { createEventDispatcher } from 'svelte';

  const dispatch = createEventDispatcher();

  function handleClick() {
    dispatch('change', { value: 42 });
    dispatch('complete'); // 无数据
  }
</script>

<button on:click={handleClick}>发送</button>
```

### 使用事件

```svelte
<Component on:change={(e) => console.log(e.detail)} />
```

### 类型化（TypeScript）

```svelte
<script lang="ts">
  const dispatch = createEventDispatcher<{
    change: { value: number };
    complete: null;
  }>();
</script>
```

> **Svelte 5 推荐**：直接用 callback props：
> `<Component onChange={(detail) => handle(detail)} />`

## Critical: Slots

`<slot>` 是 Legacy 的内容传递机制（Svelte 5 用 Snippets 替代）：

### 默认 slot

```svelte
<!-- Component.svelte -->
<div class="card">
  <slot />
</div>

<!-- 使用 -->
<Component>
  <p>内容</p>
</Component>
```

### 命名 slot

```svelte
<!-- Component.svelte -->
<slot name="footer" />
<slot name="header" />

<!-- 使用 -->
<Component>
  <p slot="header">头部</p>
  <p slot="footer">尾部</p>
</Component>
```

### Slot props（向 slot 传数据）

```svelte
<!-- Component.svelte -->
<slot value={count} onClick={handleClick} />

<!-- 使用 -->
<Component let:value let:onClick>
  <p>{value}</p>
  <button on:click={onClick}>click</button>
</Component>
```

> **注意**：Svelte 5 中，default slot props 需要 `let:` 前缀；named slots 的 slot props 不暴露给 named slots。

### $$slots（检查 slot 是否存在）

```svelte
<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();
  $: if (!$$slots.footer) {
    // footer slot 未被提供
  }
</script>
```

## Critical: svelte:component / svelte:self

### `<svelte:component>`（动态组件）

```svelte
<script>
  let Current = FirstComponent;
  $: Current = showSecond ? SecondComponent : FirstComponent;
</script>

<svelte:component this={Current} />
```

> Svelte 5 推荐：`<svelte:element this={Current} />`

### `<svelte:self>`（递归组件）

```svelte
<script>
  import { svelte:self } from 'svelte';
</script>

<svelte:self />
```

## Critical: svelte:fragment

```svelte
<!-- Component.svelte -->
<slot name="title" />
<div>
  <slot name="body" />
</div>

<!-- 使用 -->
<Component>
  <h1 slot="title">Title</h1>
  <svelte:fragment slot="body">
    <p>多元素</p>
    <p>不会被包裹</p>
  </svelte:fragment>
</Component>
```

## Critical: v5 Migration Quick Reference

| Legacy (Svelte 4) | Modern (Svelte 5) |
|---------------------|----------------------|
| `let count = 0`（顶层响应式） | `let count = $state(0)` |
| `$: x = expr` | `let x = $derived(expr)` |
| `$: { if (cond) x = v }` | `$effect(() => { if (cond) x = v })` |
| `export let prop` | `let { prop } = $props()` |
| `$$props` | `let { ...rest } = $props()` |
| `<slot />` | `{@render children()}` |
| `createEventDispatcher` | callback props |
| `on:click={fn}` | `onclick={fn}` |
| `<svelte:component this={x}>` | `<svelte:element this={x}>` |
| `<svelte:self>` | `import Self from './This.svelte'` + `<Self>` |
| `class:cool={isCool}` | `class={{ cool: isCool }}` |
| `let:slotProp={value}` | Snippets 自动传递 |

## Gotchas

1. **Legacy `$:` 基于赋值** — 数组 `.push()` 等方法不触发更新
2. **Slot props 行为不同** — default slot 可访问 slot props，named slots 不可以
3. **`createEventDispatcher` 已废弃** — Svelte 5 不推荐，新代码用 callback props
4. **`$$slots` 是编译时注入** — 可在 script 中直接使用 `$:` 中引用
5. **`<svelte:self>` 需要导入** — `import { svelte:self } from 'svelte'`

## Critical: Svelte 4 Migration (Svelte 3 → Svelte 4)

如果你正在维护 Svelte 3 项目并升级到 Svelte 4，下面这些破坏性变更必须逐项核对。详细对照与自动化脚本见 `examples/migration-v4.md` 与 `references/migration-v4-reference.md`，源码基线在 `/tmp/svelte-llms.txt:6182`。

> **推荐起点**：`npx svelte-migrate@latest svelte-4` —— 脚本会自动处理多数条目；其余需要手动介入。

### 最低版本要求

```jsonc
{
  "engines": { "node": ">=16" },
  "devDependencies": {
    "svelte": "^4.0.0",
    "@sveltejs/kit": ">=1.20.4",
    "vite-plugin-svelte": ">=2.4.1",
    "svelte-loader": ">=3.1.8",
    "rollup-plugin-svelte": ">=7.1.5",
    "typescript": ">=5"
  }
}
```

### 主要破坏性变更速查

| 类别 | Svelte 3 | Svelte 4 |
|------|----------|----------|
| **Bundler browser condition** | 可选 | Rollup `browser: true` / Webpack `conditionNames: ['browser']` |
| **CJS 输出** | 支持 | 移除（编译器、`svelte/register`、runtime） |
| **`createEventDispatcher` 类型** | 可选 payload | 严格：`T \| undefined` 可选；`T` 必填；`null` 禁 detail |
| **`Action` / `ActionReturn`** | params 隐式 `any` | 必须显式 `Action<HTMLElement, T>` |
| **`onMount` 异步清理** | 不报错 | 返回 Promise 的清理函数会被忽略（TS 报错） |
| **Custom Element** | `<svelte:options tag="x" />` | `<svelte:options customElement="x" />`（字符串或对象） |
| **`SvelteComponentTyped`** | 与 `SvelteComponent` 并存 | 弃用，统一为 `SvelteComponent` |
| **Transitions 默认范围** | 任意祖先块触发 | 仅直接父块触发，加 `\|global` 恢复旧行为 |
| **Default slot bindings** | named slot 也可见 | 仅 default slot 可见 |
| **Preprocessor 顺序** | markup×N, script×N, style×N | markup, script, style ×N（每个 preprocessor 内部三段） |
| **`eslint-plugin-svelte3`** | 推荐 | 弃用，改 `eslint-plugin-svelte` |
| **`inert` on outroing** | 不应用 | 默认应用（无障碍改进） |
| **`classList.toggle`** | 兼容老浏览器 | 需要 polyfill |
| **`CustomEvent`** | 兼容老浏览器 | 需要 polyfill |
| **`derived` falsy 值** | 警告 | 抛错 |
| **`svelte/internal` 类型** | 暴露 | 已移除 |
| **`svelte.JSX` 命名空间** | 用 | 改 `svelteHTML` / `svelte/elements` |
| **库作者 `peerDependencies`** | `^3` | `^3 \|\| ^4` |

### 自动化脚本能处理的

- `tag=` → `customElement=`（字符串形式）
- `SvelteComponentTyped` → `SvelteComponent`
- `Action` / `ActionReturn` 泛型参数
- Transitions 默认行为（添加 `|global` 修饰符）

### 需要手动处理的

- Bundler `browser` condition 配置（Rollup / Webpack）
- 严格类型错误的修复（`createEventDispatcher` payload）
- `onMount` 异步清理改同步
- Preprocessor `name` 属性 + 顺序调整
- ESLint 插件替换 + 配置迁移
- `derived` falsy 值边界条件

### Custom Element 升级

```svelte
<!-- v3 --><svelte:options tag="my-counter" />
<!-- v4 --><svelte:options customElement={{ tag: 'my-counter', props: { initial: { type: Number } } }} />
```

### onMount 异步清理反模式

```js
// ❌ v4 报错且永远不清理
onMount(async () => { const data = await fetch('/api'); setup(data); return () => teardown(); });
// ✅ 同步返回值
onMount(() => { fetch('/api').then(setup); return () => teardown(); });
```

### Preprocessor 顺序：MDsveX 必须最前

```js
// svelte.config.js
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';
import { mdsvex } from 'mdsvex';

export default {
  preprocess: [
    mdsvex({ name: 'mdsvex' }),         // 先 markup 再 script
    vitePreprocess({ name: 'vite' })
  ]
};
```

> v3：所有 markup → 所有 script → 所有 style。v4：每个 preprocessor 内部 markup → script → style，再下一个。每个 preprocessor 必须声明 `name`。

### 库作者的 peerDependencies

```jsonc
{
  "peerDependencies": { "svelte": "^3.0.0 || ^4.0.0" },
  "devDependencies": { "svelte": "^4.0.0" }
}
```

### Run-it-Yourself Checklist

```bash
npm install svelte@^4                    # 1. 升级依赖
npx svelte-migrate@latest svelte-4       # 2. 跑迁移脚本
npx svelte-check                          # 3. 看 TypeScript
npm test                                   # 4. 跑测试
```

如大量出现 `Action` 泛型错误，说明 migration 脚本漏掉；手动补 `<HTMLElement, YourType>`。

> 进一步 v5 迁移在 svelte-runes / svelte-misc 中；本技能只覆盖 Svelte 4 升级。

## Quick Fixes

| 问题 | 解决 |
|------|------|
| 数组 `.push()` 不更新 | `items = [...items, newItem]` 或 `items = items` |
| slot 样式不生效 | 样式隔离 scoped，不影响 slot 内容 |
| `$$restProps` 类型错误 | 确认是 `export let` 而非 `let` |
| EventDispatcher 类型问题 | 升级 TypeScript 到 5+ |

## FAQ

**Q: 什么时候必须用 Legacy API？**
A: 维护未迁移的 Svelte 4 项目时。新项目应使用 Runes。

**Q: 可以混合使用 Legacy 和 Runes 吗？**
A: 可以。一个组件使用 `$state` 即进入 Runes Mode，未使用的部分用 Legacy 语法。

**Q: `$:` 和 `$derived` 的区别？**
A: 功能相似，但 `$derived` 是 Svelte 5 的 Runes API，更显式和可控。`$:` 在 Legacy Mode 可用。

**Q: Slot 和 Snippet 哪个更好？**
A: Snippets 更强大灵活。Slot 适合简单内容传递；Snippet 可接收参数、可在定义处使用、类型安全更好。

## Cross-References

本技能是 Svelte Legacy 维护专用，与以下技能协同：

- Svelte 5 入口与学习路径 → `../svelte-awesome/`
- Svelte 4 → Svelte 5 迁移对照 → `../svelte-misc/`（migration 章节）
- Runes 响应式（`$state` / `$derived` / `$effect`）→ `../svelte-runes/`
- Stores（writable / readable / derived）→ `../svelte-lifecycle/`
- 模板语法（snippet / event / bind）→ `../svelte-template-syntax/`
- 调试响应式不更新 / `$inspect` → `../svelte-lifecycle/`
- 把组件编译为 Web Component → `../svelte-misc/`（Custom Elements 章节）
- 初始化与迁移脚本 → `../svelte-cli/`（`sv migrate svelte-5`）

## Examples

Practical examples for working with Legacy APIs:

- [examples/README.md](./examples/README.md) - Examples entry point
- [examples/migration-scripts.md](./examples/migration-scripts.md) - Migration workflows (svelte-migrate, let→$state, $:→$derived, etc.)
- [examples/legacy-mode.md](./examples/legacy-mode.md) - Legacy mode usage, mixing legacy and runes
- [examples/store-comparison.md](./examples/store-comparison.md) - Store patterns in Svelte 4 and Svelte 5
- [examples/migration-v4.md](./examples/migration-v4.md) - Svelte 3 → Svelte 4 迁移 12 个实战示例（bundler 配置、严格类型、Custom Element、transition、slot、preprocessor 等）

## References

Complete reference documentation:

- [references/README.md](./references/README.md) - References entry point
- [references/migration-reference.md](./references/migration-reference.md) - Complete migration reference with before/after code
- [references/legacy-mode-reference.md](./references/legacy-mode-reference.md) - Legacy mode behavior and limitations
- [references/store-reference.md](./references/store-reference.md) - Store API reference (writable, readable, derived, custom stores)
- [references/migration-v4-reference.md](./references/migration-v4-reference.md) - Svelte 3 → Svelte 4 完整对照参考（最低版本、bundler、严格类型、Custom Element、slot、preprocessor、其他 breaking change、库作者指引）
