---
name: svelte-reference
description: Svelte 5 API 参考技能。当用户需要查阅 svelte/action、svelte/store、svelte/transition、svelte/easing、svelte/compiler、svelte/events 的具体 API，或需要查看编译器错误/警告代码时使用。
---

# Svelte API Reference (Svelte 5)

本技能是 Svelte 官方模块 API 的快速参考，包括 `svelte/store`、`svelte/action`、`svelte/transition`、`svelte/easing`、`svelte/events`、`svelte/compiler` 及常见编译器/运行时错误代码。

## When to use this skill

当用户需要查阅 Svelte 具体模块的 API 签名、使用 `transition`、`animate`、`easing` 函数、错误代码含义，或需要编译器配置时使用本技能。

## Critical: svelte/store

```js
import { writable, readable, derived, readonly, get } from 'svelte/store';
```

| 函数 | 签名 | 说明 |
|------|------|------|
| `writable` | `(initial, start?) => { subscribe, set, update }` | 可写 store |
| `readable` | `(initial, start?) => { subscribe }` | 只读 store |
| `derived` | `(stores, fn, initial?) => { subscribe }` | 派生 store |
| `readonly` | `(store) => store` | 返回只读版本 |
| `get` | `(store) => value` | 非订阅获取值（同步） |

### Store Contract

```ts
store = {
  subscribe: (subscription: (value: any) => void) => (() => void),
  set?: (value: any) => void
}
```

## Critical: svelte/action

Action 是返回 `{ destroy?: () => void }` 的函数：

```svelte
<script>
  import { onMount } from 'svelte';

  function myAction(node, param) {
    // node: HTMLElement
    // param: any (从 use:action(param) 传入)
    return {
      update(newParam) { /* param 变化时调用 */ },
      destroy() { /* 卸载时调用 */ }
    };
  }
</script>

<div use:myAction={42}>...</div>
```

**Svelte 5 推荐**：`{@attach ...}` 替代 action（更好的依赖追踪）。

## Critical: svelte/transition

```svelte
<script>
  import { fade, fly, slide, scale, draw, crossfade } from 'svelte/transition';
</script>

{#if visible}
  <div transition:fade={{ duration: 300, delay: 100 }}>fade</div>
  <div transition:fly={{ y: 20, duration: 300 }}>fly</div>
  <div transition:slide>slide</div>
  <div transition:scale={{ opacity: 0, duration: 300 }}>scale</div>
{/if}
```

### 内置过渡

| 函数 | 效果 |
|------|------|
| `fade` | 透明度 |
| `fly` | 位置飞入 |
| `slide` | 滑动 |
| `scale` | 缩放 |
| `draw` | SVG 路径绘制 |
| `crossfade` | 成对淡入淡出 |

### 过渡参数

```ts
{
  delay?: number,      // 开始前延迟 (ms)
  duration?: number,    // 持续时间 (ms)
  easing?: function,    // 缓动函数
  css?: (t, u) => string,  // 自定义 CSS
  tick?: (t, u) => void    // 自定义 JS
}
```

### 全局过渡

```svelte
<!-- 跨控制流块都触发 -->
<div transition:fade|global>...</div>

<!-- 本地（默认） -->
<div transition:fade>...</div>
```

## Critical: svelte/easing

```js
import { cubicIn, cubicOut, cubicInOut, elasticOut, bounceOut } from 'svelte/easing';
```

| 缓动 | 说明 |
|------|------|
| `linear` | 匀速 |
| `cubicIn`/`cubicOut`/`cubicInOut` | 三次缓动 |
| `quadIn`/`quadOut`/`quadInOut` | 二次 |
| `quartIn`/`quartOut`/`quartInOut` | 四次 |
| `quintIn`/`quintOut`/`quintInOut` | 五次 |
| `sineIn`/`sineOut`/`sineInOut` | 正弦 |
| `elasticIn`/`elasticOut`/`elasticInOut` | 弹性 |
| `bounceIn`/`bounceOut`/`bounceInOut` | 弹跳 |

```svelte
<div transition:fly={{ y: 100, easing: cubicOut }}>...</div>
```

## Critical: svelte/animate

```svelte
<script>
  import { flip } from 'svelte/animate';
</script>

{#each items as item (item.id)}
  <div animate:flip>{item.name}</div>
{/each}
```

### flip 参数

```ts
{
  delay?: number,
  duration?: number | (node, { from, to }) => number,
  easing?: function
}
```

## Critical: svelte/events

提供 `on` 函数，替代 `addEventListener`（正确处理事件委托和 `stopPropagation`）：

```svelte
<script>
  import { on } from 'svelte/events';

  const handler = on(node, 'click', (event) => {
    // 正确处理委托和 stopPropagation
  }, { passive: true });
</script>
```

## Critical: svelte/reactivity

### $effect.tracking

```svelte
$effect.tracking(); // boolean，是否在追踪上下文中
```

### createSubscriber

创建订阅者（只在追踪上下文中工作）：

```svelte
import { createSubscriber } from 'svelte/reactivity';
const subscriber = createSubscriber((track) => {
  track(value); // 声明依赖
});
```

## Critical: Compiler Errors (常见)

| 代码 | 含义 | 解决 |
|------|------|------|
| `2304` | 未定义的引用 | 检查变量名拼写和导入 |
| `2322` | 目标元素不存在 | 检查 `target` selector |
| `2554` | 类型不匹配 | 检查 TypeScript 类型标注 |
| `2451` | 在非 effect 中修改 state | 不要在 `$effect` 中直接赋值 `$state` |
| `7006` | 参数类型错误 | 检查函数签名 |

## Critical: Runtime Warnings (常见)

| 代码 | 含义 | 解决 |
|------|------|------|
| `ownership_invalid_mutation` | 修改了非 own 的状态 | 不要直接修改 prop， 用 `$bindable` |
| `await_waterfall` | await 依赖链导致性能问题 | 重构数据获取逻辑 |
| `state_teardown` | 尝试在销毁后更新 | 确保组件未卸载后访问 |

## Critical: svelte/compiler

### 配置项

```js
import { compile } from 'svelte/compiler';
const result = compile(source, {
  filename: 'App.svelte',
  generate: 'dom',       // 'dom' | 'ssr' | false
  name: 'Component',      // 导出名
  format: 'esm',         // 'esm' | 'cjs'
  sveltePath: 'svelte',  // svelte 包路径
  dev: true,             // 开发模式（添加检查）
  css: 'injected',       // CSS 注入方式
  runes: false,           // 强制 Legacy Mode
  // ...
});
```

### compile 输出

```js
result.js.code;      // 编译后的 JS
result.css.code;     // 编译后的 CSS（如有）
result.warnings;     // 警告列表
result.stats;        // 编译统计
```

## Quick Fixes

| 问题 | 解决 |
|------|------|
| 过渡不触发 | 确认条件块变化时元素被销毁/重建；用 `{#key}` |
| `flip` 动画不流畅 | 确保 key 唯一且稳定 |
| 缓动函数不工作 | 确认从 `svelte/easing` 正确导入 |
| 编译器报错 | 查看上方错误代码表 |
| 运行时警告 | 检查 ownership 链 |

## Examples

Practical usage patterns with detailed explanations:

- [examples/README.md](./examples/README.md) - Entry point
- [examples/store-patterns.md](./examples/store-patterns.md) - Writable, readable, derived stores, auto-subscribe, cross-component sharing
- [examples/motion.md](./examples/motion.md) - Tweened and spring animations with easing
- [examples/transition.md](./examples/transition.md) - Built-in transitions, custom transitions, in/out transitions

## References

Complete API documentation:

- [references/README.md](./references/README.md) - Entry point
- [references/store-reference.md](./references/store-reference.md) - Store API signatures: writable, readable, derived, get
- [references/motion-reference.md](./references/motion-reference.md) - Motion API: tweened, spring, easing functions
- [references/transition-reference.md](./references/transition-reference.md) - Transition API: fade, fly, slide, scale, custom transitions

## Gotchas

1. **`svelte/transition` 的 `css` 和 `tick` 互斥** — 两者只能用一个
2. **`flip` 需要稳定 key** — 用唯一 ID 而非数组索引
3. **Store subscribe 返回 unsubscribe 函数** — 组件卸载时自动取消订阅
4. **`get(store)` 会创建临时订阅** — 频繁调用影响性能
5. **`elasticIn/elasticOut` 较慢** — 大量使用可能影响性能
