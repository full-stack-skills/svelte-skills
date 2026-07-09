---
name: svelte-reference
description: Svelte 5 API 参考技能。当用户需要查阅 svelte/action、svelte/store、svelte/transition、svelte/animate、svelte/easing、svelte/compiler、svelte/events 的具体 API，或需要查看编译器错误/警告代码时使用。
---

# Svelte API Reference (Svelte 5)

本技能是 Svelte 官方模块 API 的快速参考，包括 `svelte/store`、`svelte/action`、`svelte/transition`、`svelte/animate`、`svelte/easing`、`svelte/events`、`svelte/compiler` 及常见编译器/运行时错误代码。

## When to use this skill

当用户需要查阅 Svelte 具体模块的 API 签名、使用 `transition` / `in` / `out` / `animate` 指令、`use:` action、`easing` 函数、错误代码含义，或需要编译器配置时使用本技能。

## Critical: svelte/store

```js
import { writable, readable, derived, readonly, get } from 'svelte/store';
```

| 函数 | 签名 | 说明 |
|------|------|------|
| `writable` | `(initial, start?) => { subscribe, set, update }` | 可写 store |
| `readable` | `(initial, start) => { subscribe }` | 只读 store（值只能由内部 `start` 设置） |
| `derived` | `(stores, fn, initial?) => { subscribe }` | 派生 store |
| `readonly` | `(store) => store` | 返回只读版本（无 `set`/`update`） |
| `get` | `(store) => value` | 非订阅获取值（同步，创建临时订阅） |

### Store Contract

```ts
store = {
  subscribe: (subscription: (value: any) => void) => (() => void),
  set?: (value: any) => void
}
```

### When to Use Stores vs Runes

- **Prefer runes** (`$state` in `.svelte.js` files) for shared state and logic extraction.
- **Use stores** when you have complex async data streams, need manual control over updates, or want RxJS-style operators.

### `writable` Start Function

`writable(initial, start)` — the `start` callback is called when the subscriber count goes `0 → 1` (not `1 → 2`). Return a `stop` function that runs when subscribers go `1 → 0`. The `start` callback also receives an `update` function (in addition to `set`) for transform-based updates.

### `derived` Forms

```js
// single store
const doubled = derived(a, ($a) => $a * 2);

// array of stores
const sum = derived([a, b], ([$a, $b]) => $a + $b);

// async — second arg is `set`, third is `update`, third `derived` arg is initial value
const delayed = derived(a, ($a, set) => {
  setTimeout(() => set($a), 1000);
}, 0);

// with cleanup — return a function from the callback
const tick = derived(frequency, ($frequency, set) => {
  const id = setInterval(() => set(Date.now()), 1000 / $frequency);
  return () => clearInterval(id);
}, 0);
```

### `get(store)` Caveat

Creates a temporary subscription, reads, then unsubscribes. Avoid in hot paths.

## Critical: svelte/action (use: directive)

Actions are functions called when an element is mounted, added with the `use:` directive:

```svelte
<script>
  /** @type {import('svelte/action').Action} */
  function myAction(node) {
    // node: HTMLElement — runs once on mount (client only, not SSR)
    $effect(() => {
      // setup
      return () => {
        // teardown — runs on unmount
      };
    });
  }
</script>

<div use:myAction>...</div>
```

> [!NOTE]
> In Svelte 5.29 and newer, consider using [`{@attach ...}`](@attach) instead — more flexible and composable.

### Action With Parameter

The parameter is **not reactive** — the action runs once on mount. Use `$effect` inside to react to parameter changes.

```svelte
<script>
  /** @type {import('svelte/action').Action<HTMLInputElement, number>} */
  function debounce(node, delay) {
    $effect(() => {
      const handler = () => setTimeout(() => node.dispatchEvent(new Event('done')), delay);
      node.addEventListener('input', handler);
      return () => node.removeEventListener('input', handler);
    });
  }
</script>

<input use:debounce={300} />
```

### Legacy `update`/`destroy` Form

```svelte
<script>
  /** @type {import('svelte/action').Action<HTMLDivElement, string>} */
  function colorize(node, color) {
    node.style.color = color;
    return {
      update(newColor) { node.style.color = newColor; },
      destroy() { node.style.color = ''; }
    };
  }
</script>

<div use:colorize={'red'}>...</div>
```

Prefer the `$effect` form for new code.

### Typing — `Action<Node, Parameter, Events>`

```svelte
<script lang="ts">
  import type { Action } from 'svelte/action';

  const gestures: Action<HTMLDivElement, undefined, {
    onswipeleft: (e: CustomEvent) => void;
    onswiperight: (e: CustomEvent) => void;
  }> = (node) => {
    // dispatch: node.dispatchEvent(new CustomEvent('swipeleft'));
  };
</script>

<div use:gestures onswipeleft={next} onswiperight={prev}>...</div>
```

The third type parameter makes `onswipeleft` / `onswiperight` type-check in the template.

## Critical: svelte/transition (transition: directive)

```svelte
<script>
  import { fade, fly, slide, scale, blur, draw, crossfade } from 'svelte/transition';
</script>

{#if visible}
  <div transition:fade={{ duration: 300, delay: 100 }}>fade</div>
  <div transition:fly={{ y: 20, duration: 300 }}>fly</div>
  <div transition:slide>slide</div>
  <div transition:scale={{ start: 0.5, duration: 300 }}>scale</div>
  <div transition:blur={{ amount: 10 }}>blur</div>
{/if}
```

`transition:` is **bidirectional** — reverses on interrupt.

### Built-in Transitions

| 函数 | 效果 | 关键参数 |
|------|------|----------|
| `fade` | 透明度 | `duration` |
| `fly` | 位置飞入 | `x`, `y`, `opacity` |
| `slide` | 滑动 | `axis: 'x' \| 'y'` |
| `scale` | 缩放 | `start`, `opacity` |
| `blur` | 模糊 + 透明度 | `amount` |
| `draw` | SVG 路径绘制 | `speed` or `duration` |
| `crossfade` | 成对淡入淡出 | `duration`, `fallback` |

### Transition Parameters

```ts
{
  delay?: number,        // ms before starting (default 0)
  duration?: number,     // ms (default 400)
  easing?: function,     // easing (default linear)
  css?: (t, u) => string,// web animation CSS (preferred)
  tick?: (t, u) => void  // imperative (use only when CSS cannot)
}
```

`t` runs `0 → 1` on intro, `1 → 0` on outro. `u = 1 - t`. Easing is applied to `t` before it reaches `css` / `tick`. **`css` and `tick` are mutually exclusive.**

### Local vs Global

```svelte
<!-- local: only animates when this block is created/destroyed -->
<div transition:fade>...</div>

<!-- global: animates whenever ANY ancestor block toggles -->
<div transition:fade|global>...</div>
```

### Transition Events

| Event | Fires |
|-------|-------|
| `introstart` | before intro animation |
| `introend` | after intro animation |
| `outrostart` | before outro animation |
| `outroend` | after outro animation |

```svelte
<div
  transition:fly={{ y: 200, duration: 2000 }}
  onintrostart={() => (status = 'intro started')}
  onoutroend={() => (status = 'outro ended')}
>...</div>
```

### Custom Transition Function

```ts
// @noErrors
transition = (
  node: HTMLElement,
  params: any,
  options: { direction: 'in' | 'out' | 'both' }
) => {
  delay?: number;
  duration?: number;
  easing?: (t: number) => number;
  css?: (t: number, u: number) => string;
  tick?: (t: number, u: number) => void;
}
```

If the function returns a **function** (instead of an object), Svelte calls it in the next microtask — this is what `crossfade` uses to coordinate paired transitions.

## Critical: in: and out: (separate transitions)

`in:` and `out:` are **unidirectional** — they do **not** reverse on interrupt. If you toggle a block off mid-intro, the in-flight intro is **abandoned** and the out-transition restarts from `t=0`.

```svelte
<script>
  import { fade, fly } from 'svelte/transition';
  let visible = $state(false);
</script>

{#if visible}
  <!-- flies in, fades out — NOT a reversible fly/fade -->
  <div in:fly={{ y: 200 }} out:fade>...</div>
{/if}
```

| Directive | When it runs | Reversible on interrupt? |
|-----------|--------------|--------------------------|
| `transition:` | enter or leave | **yes** (reverses mid-flight) |
| `in:` | enter only | **no** (abandoned) |
| `out:` | leave only | **no** (always plays forward) |

Use `in:` / `out:` when enter and exit animations should be visually different. The third argument to a custom transition function is `{ direction: 'in' | 'out' | 'both' }`.

## Critical: svelte/easing

```js
import { cubicIn, cubicOut, cubicInOut, elasticOut, bounceOut } from 'svelte/easing';
```

| 缓动 | 说明 |
|------|------|
| `linear` | 匀速 |
| `quadIn`/`quadOut`/`quadInOut` | 二次 |
| `cubicIn`/`cubicOut`/`cubicInOut` | 三次缓动 |
| `quartIn`/`quartOut`/`quartInOut` | 四次 |
| `quintIn`/`quintOut`/`quintInOut` | 五次 |
| `sineIn`/`sineOut`/`sineInOut` | 正弦 |
| `expoIn`/`expoOut`/`expoInOut` | 指数 |
| `circIn`/`circOut`/`circInOut` | 圆 |
| `elasticIn`/`elasticOut`/`elasticInOut` | 弹性 |
| `backIn`/`backOut`/`backInOut` | 回退 |
| `bounceIn`/`bounceOut`/`bounceInOut` | 弹跳 |

```svelte
<div transition:fly={{ y: 100, easing: cubicOut }}>...</div>
```

## Critical: animate: (svelte/animate)

`animate:` runs when the **index of an existing item changes** inside a keyed `{#each}` block — not on add/remove. It must be on the **immediate child** of a keyed each block.

```svelte
<script>
  import { flip } from 'svelte/animate';
  let items = $state(['apple', 'banana', 'cherry']);
</script>

{#each items as item (item)}
  <li animate:flip={{ duration: 400, easing: quintOut }}>{item}</li>
{/each}
```

### `flip` (FLIP technique)

Measures `from` and `to` bounding rects, applies an inverse `transform`, then animates back to identity. See [references/animations-reference.md](./references/animations-reference.md) for full API.

```ts
flip(node, {
  delay?: number,
  duration?: number | (node, { from, to }) => number,
  easing?: function
})
```

### Custom Animation Function

```ts
// @noErrors
animation = (
  node: HTMLElement,
  { from: DOMRect, to: DOMRect },
  params: any
) => {
  delay?: number;
  duration?: number;
  easing?: (t: number) => number;
  css?: (t: number, u: number) => string;
  tick?: (t: number, u: number) => void;
}
```

`from` and `to` are the element's bounding rects before and after the reorder. Compute distance, angle, etc. to drive custom effects:

```svelte
<script>
  import { cubicOut } from 'svelte/easing';

  function whizz(node, { from, to }) {
    const dx = from.left - to.left;
    const dy = from.top - to.top;
    const d = Math.sqrt(dx * dx + dy * dy);
    return {
      duration: Math.sqrt(d) * 120,
      easing: cubicOut,
      css: (t, u) => `transform: translate(${u * dx}px, ${u * dy}px) rotate(${t * 360}deg);`
    };
  }
</script>

{#each list as item (item)}
  <div animate:whizz>{item}</div>
{/each}
```

`animate:` does **not** run on add/remove — combine with `in:` / `out:` for those.

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
| `in:`/`out:` 中途切换行为奇怪 | 这是 unidirectional 的预期行为 — `transition:` 才可逆 |
| `animate:flip` 不工作 | 必须有 `(key)` 且 `animate:` 必须在 keyed each 的**直接子元素**上 |
| `flip` 动画不流畅 | 确保 key 唯一且稳定 |
| 缓动函数不工作 | 确认从 `svelte/easing` 正确导入 |
| `use:` 参数变化不生效 | action 只调用一次；用 `$effect` 内部读取参数 |
| 编译器报错 | 查看上方错误代码表 |
| 运行时警告 | 检查 ownership 链 |

## Examples

Practical usage patterns with detailed explanations:

- [examples/README.md](./examples/README.md) - Entry point
- [examples/store-patterns.md](./examples/store-patterns.md) - Writable, readable, derived stores, auto-subscribe, cross-component sharing
- [examples/motion.md](./examples/motion.md) - Tweened and spring animations with easing
- [examples/transition.md](./examples/transition.md) - Built-in transitions, custom transitions, in/out transitions
- [examples/transitions-advanced.md](./examples/transitions-advanced.md) - Crossfade, deferred transitions, transition events, custom css/tick
- [examples/in-out-animate.md](./examples/in-out-animate.md) - in:/out: separate transitions, animate:flip, custom animation functions
- [examples/use-action.md](./examples/use-action.md) - use: directive with typing, lifecycle, custom events

## References

Complete API documentation:

- [references/README.md](./references/README.md) - Entry point
- [references/store-reference.md](./references/store-reference.md) - Store API signatures: writable, readable, derived, get, readonly
- [references/motion-reference.md](./references/motion-reference.md) - Motion API: tweened, spring, easing functions
- [references/transition-reference.md](./references/transition-reference.md) - Transition API: fade, fly, slide, scale, custom transitions
- [references/transitions-complete.md](./references/transitions-complete.md) - Complete reference for all built-in transitions, events, |global, custom functions
- [references/in-out-reference.md](./references/in-out-reference.md) - in: / out: separate (unidirectional) transitions
- [references/animations-reference.md](./references/animations-reference.md) - animate: directive, flip, custom animation functions
- [references/use-action-reference.md](./references/use-action-reference.md) - use: action typing, lifecycle, Action<>, custom events

## Gotchas

1. **`svelte/transition` 的 `css` 和 `tick` 互斥** — 两者只能用一个
2. **`in:` / `out:` 不可逆** — 中断时 in-flight intro 被放弃，outro 从 t=0 重启
3. **`animate:` 必须在 keyed each 的直接子元素** — 嵌套 div 上的 animate: 不会运行
4. **`flip` 需要稳定 key** — 用唯一 ID 而非数组索引
5. **`animate:` 不响应 add/remove** — 只在 reorder 时触发
6. **Store subscribe 返回 unsubscribe 函数** — 组件卸载时自动取消订阅
7. **`get(store)` 会创建临时订阅** — 频繁调用影响性能
8. **`elasticIn/elasticOut` 较慢** — 大量使用可能影响性能
9. **`use:action` 参数变化不会重新调用** — 用 `$effect` 在 action 内部响应
