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

## Examples

Practical examples for working with Legacy APIs:

- [examples/README.md](./examples/README.md) - Examples entry point
- [examples/migration-scripts.md](./examples/migration-scripts.md) - Migration workflows (svelte-migrate, let→$state, $:→$derived, etc.)
- [examples/legacy-mode.md](./examples/legacy-mode.md) - Legacy mode usage, mixing legacy and runes
- [examples/store-comparison.md](./examples/store-comparison.md) - Store patterns in Svelte 4 and Svelte 5

## References

Complete reference documentation:

- [references/README.md](./references/README.md) - References entry point
- [references/migration-reference.md](./references/migration-reference.md) - Complete migration reference with before/after code
- [references/legacy-mode-reference.md](./references/legacy-mode-reference.md) - Legacy mode behavior and limitations
- [references/store-reference.md](./references/store-reference.md) - Store API reference (writable, readable, derived, custom stores)
