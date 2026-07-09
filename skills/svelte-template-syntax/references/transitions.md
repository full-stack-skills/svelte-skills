# transitions / in / out Reference

## 触发条件

**Transitions** 在元素进入/离开 DOM 时触发（如 `{#if}` / `{#each}` / `{#key}` 状态变化）。

> 块（如 `{#if}`）卸载时，块内所有元素（含无 transition 的）保留在 DOM 中直到**所有** transition 完成。

## 指令类型

| 指令 | 行为 |
|------|------|
| `transition:` | 双向 — 状态反转时回滚（reverse） |
| `in:` | 单向入场 |
| `out:` | 单向出场 |

`in:` 与 `out:` 同时触发时**并行播放**，互不打断；`transition:` 在中途反转时回滚。

```svelte
<script>
  import { fade, fly } from 'svelte/transition';
  let visible = $state(false);
</script>

{#if visible}
  <div in:fly={{ y: 200 }} out:fade>flies in, fades out</div>
{/if}
```

## 局部 vs 全局

```svelte
{#if x}
  {#if y}
    <p transition:fade>仅 y 变化时</p>
    <p transition:fade|global>x 或 y 变化时</p>
  {/if}
{/if}
```

`transition:foo` 默认是**局部**的，仅响应**所属块**的创建/销毁。`|global` 修饰符让它响应祖先块。

## 内置 transitions（`svelte/transition`）

| 名称 | 描述 | 参数 |
|------|------|------|
| `fade` | 透明度 | `duration`, `delay`, `easing`, `opacity` |
| `blur` | 模糊 + 透明度 | `amount`, `duration`, `delay`, `easing` |
| `fly` | 位移 | `x`, `y`, `duration`, `delay`, `easing` |
| `slide` | 水平/垂直滑动 | `duration`, `delay`, `easing` |
| `scale` | 缩放 | `start`, `opacity`, `duration`, `delay`, `easing` |
| `draw` | SVG 描边 | `duration`, `delay`, `easing` |
| `crossfade` | 元素间交叉 | 需 `send`/`receive` |

```svelte
import { fade, fly, slide, scale, blur, draw } from 'svelte/transition';
```

## Transition 参数

```svelte
<div transition:fade={{ duration: 2000, delay: 200 }}>...</div>
```

通用字段：
- `delay: number` — 延迟毫秒
- `duration: number` — 毫秒
- `easing: (t: number) => number` — `0..1 → 0..1`

特定字段：
- `fade.opacity: number` — 起始透明度（默认 0）
- `fly.x: number` / `fly.y: number` — 位移
- `scale.start: number` — 起始 scale（默认 0）
- `blur.amount: number` — 模糊 px（默认 5）
- `slide.axis: 'x' | 'y'` — 滑动轴

## 自定义 transition 函数

```ts
type TransitionFn = (
  node: HTMLElement,
  params: any,
  options: { direction: 'in' | 'out' | 'both' }
) => TransitionConfig | (() => TransitionConfig);

interface TransitionConfig {
  delay?: number;
  duration?: number;
  easing?: (t: number) => number;
  css?: (t: number, u: number) => string;
  tick?: (t: number, u: number) => void;
}
```

### css vs tick

- **`css(t, u)`** → 返回 CSS 字符串；Svelte 生成 [Web Animations](https://developer.mozilla.org/en-US/docs/Web/API/Web_Animations_API)，可跑在主线程外，性能更好
- **`tick(t, u)`** → 在 JS 中手动操作；每帧回调，仅在 css 不可用时使用

参数语义：
- `t` — `0..1` 已应用 easing
- `u` — `1 - t`
- 入场：t 从 0 → 1；出场：t 从 1 → 0
- `1` = 元素自然状态

### tick 模式（需要时使用）

```svelte
<script>
  function typewriter(node, { speed = 1 }) {
    const text = node.textContent;
    const duration = text.length / (speed * 0.01);

    return {
      duration,
      tick: (t) => {
        const i = ~~(text.length * t);
        node.textContent = text.slice(0, i);
      }
    };
  }
</script>

<p in:typewriter={{ speed: 1 }}>The quick brown fox...</p>
```

### 异步初始化（crossfade）

返回函数而非对象 → 在下一个 microtask 调用 → 多个 transition 可协调（crossfade 效果）：

```js
function myFade(node, params) {
  return () => ({ duration: 400, css: t => `opacity: ${t}` });
}
```

## options.direction

`options.direction` 标识 transition 是 `in` / `out` / `both`。`transition:` 触发时为 `both`；`in:` / `out:` 分别为 `in` / `out`。

可在自定义函数中根据方向调整行为。

## Transition 事件

```svelte
<p
  transition:fly={{ y: 200 }}
  onintrostart={...}
  onoutrostart={...}
  onintroend={...}
  onoutroend={...}
>...</p>
```

| 事件 | 触发时机 |
|------|----------|
| `introstart` | 入场开始 |
| `introend` | 入场结束 |
| `outrostart` | 出场开始 |
| `outroend` | 出场结束 |

## 跨元素 crossfade

```svelte
<script>
  import { crossfade } from 'svelte/transition';

  const [send, receive] = crossfade({
    duration: 400,
    fallback(node) {
      const style = getComputedStyle(node);
      const transform = style.transform === 'none' ? '' : style.transform;
      return {
        duration: 400,
        css: t => `transform: ${transform} scale(${t})`
      };
    }
  });
</script>

{#each items as item (item.id)}
  <li in:receive={{ key: item.id }} out:send={{ key: item.id }}>
    {item.text}
  </li>
{/each}
```

`crossfade` 返回 `[send, receive]` — `out:send` 的元素和 `in:receive` 元素配对（同 key）。

## 与 {#key} 配合

```svelte
{#key value}
  <div transition:fade>{value}</div>
{/key}
```

每次 `value` 变化触发 intro + outro（即使值不空）。

## 列表 transitions

```svelte
{#each items as item (item.id)}
  <div transition:slide>
    {item.text}
  </div>
{/each}
```

- 新增：触发 `in:`
- 删除：触发 `out:`
- 重排：不触发（用 `animate:`）

## 性能与限制

- 优先用 `css` 而非 `tick`（Web Animations 离主线程）
- 避免在 100+ 元素列表上同时跑复杂 transition
- `transition:fade={{ duration: 0 }}` 不可用 — duration 至少 1
- transition 在 SSR 期间**不**运行
- 同元素上 transition 和 in/out 不能混用

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| transition 不触发 | 父级总是重建 | 用 `{#each}` keyed 或拆解 |
| 反向不工作 | 用的是 `in:` / `out:` | 改 `transition:` |
| 类型错误 `direction` | 函数未接收 options | `(node, params, options) =>` |
| 元素不消失 | block 还在 | 确认条件已变 false |
| fade 闪烁 | `opacity: 0` 默认值 | 调整 `opacity` 参数 |
