# $effect 深层机制

## 概述

`$effect(fn)` 在组件**挂载后**和**依赖变化后（microtask）**运行函数。
- 仅在浏览器中执行（SSR 自动跳过）
- 用于副作用：DOM 操作、第三方库调用、网络请求
- **不应**用于同步状态（用 `$derived` 替代）

## 生命周期

```
component mount
  ↓
$effect 首次执行（同步读取建立依赖）
  ↓
state change → 标脏，microtask 调度
  ↓
DOM 更新
  ↓
effect 重跑（teardown → re-run）
  ↓
component destroy → teardown
```

文档原文："Your effects run after the component has been mounted to the DOM, and in a microtask after state changes. Re-runs are batched (i.e. changing color and size in the same moment won't cause two separate runs), and happen after any DOM updates have been applied."

## 基本执行时机

```svelte
<script>
  let count = $state(0);

  $effect(() => {
    // 1. 挂载后立即
    // 2. count 变化时重跑
    console.log('effect ran, count =', count);
  });
</script>
```

> `$effect` 可在组件任意位置调用，**只要在某个父 effect 运行期间**。Svelte 用 effect 内部表示模板逻辑。

## Cleanup / Teardown

```svelte
<script>
  let enabled = $state(true);
  let tick = $state(0);

  $effect(() => {
    if (!enabled) return;

    const id = setInterval(() => tick += 1, 1000);

    return () => {
      // (a) 父 effect 重跑前
      // (b) 父级销毁时
      clearInterval(id);
    };
  });
</script>
```

> 文档："a) immediately before the effect re-runs b) when the component is destroyed"

## 依赖追踪

### 同步读取追踪

```svelte
<script>
  let color = $state('#ff3e00');
  let size = $state(50);

  $effect(() => {
    const ctx = canvas.getContext('2d');
    ctx.fillStyle = color;       // 追踪
    ctx.fillRect(0, 0, size, size); // 追踪
  });
</script>
```

### 异步读取不追踪

```ts
$effect(() => {
  const ctx = canvas.getContext('2d');
  ctx.fillStyle = color;            // 追踪

  setTimeout(() => {
    ctx.fillRect(0, 0, size, size); // 不追踪
  }, 0);
});
```

> 文档："Values that are read asynchronously — after an await or inside a setTimeout, for example — will not be tracked."

### 对象整体 vs 内部属性

```svelte
<script>
  let state = $state({ value: 0 });

  // 仅当 state 整体重新赋值才重跑
  $effect(() => { state; });

  // 每次 state.value 变化都重跑
  $effect(() => { state.value; });
</script>
```

### 条件分支决定依赖

effect **只追踪上次运行时实际读取的状态**：

```ts
import confetti from 'canvas-confetti';

let condition = $state(true);
let color = $state('#ff3e00');

$effect(() => {
  if (condition) {
    confetti({ colors: [color] });
  } else {
    confetti();
  }
});
// 切到 false → color 不再是依赖；切回 true → 重新追踪
```

### 间接调用也算

```ts
function helper() { return count; } // 同步读
$effect(() => { helper(); });       // count 是依赖
```

### 类中 $effect 不追踪

```svelte
<script>
  class Logger {
    count = $state(0);
    log = $effect(() => {
      // 这里读 this.count 不会建立依赖
      console.log(this.count);
    });
  }
</script>
```

> 文档："If `$state` and `$derived` are used directly inside the `$effect` (for example, during creation of a reactive class), those values will not be treated as dependencies."

## $effect.pre（DOM 更新前）

```svelte
<script>
  import { tick } from 'svelte';
  let div: HTMLDivElement;
  let messages = $state<string[]>([]);

  $effect.pre(() => {
    if (!div) return; // 挂载前跳过
    messages.length;  // 显式依赖
    if (div.offsetHeight + div.scrollTop > div.scrollHeight - 20) {
      tick().then(() => {
        div.scrollTo(0, div.scrollHeight);
      });
    }
  });
</script>
```

除时序外与 `$effect` 完全相同。

## $effect.tracking（判断上下文）

```svelte
<script>
  console.log('setup:', $effect.tracking()); // false

  $effect(() => {
    console.log('in effect:', $effect.tracking()); // true
  });
</script>

<p>in template: {$effect.tracking()}</p> <!-- true -->
```

> 用于实现 `createSubscriber` 这类工具 —— 仅在状态被追踪时订阅，事件处理器中不订阅。

## $effect.pending()（待定 Promise 数量）

Svelte 5.x 引入。返回**当前 boundary**（不含子 boundary）中待定的 Promise 数量：

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  async function add(a, b) {
    await new Promise(f => setTimeout(f, 500));
    return a + b;
  }
</script>

<button onclick={() => a++}>a++</button>
<button onclick={() => b++}>b++</button>

<p>{a} + {b} = {await add(a, b)}</p>

{#if $effect.pending()}
  <span class="spinner">pending: {$effect.pending()}</span>
{/if}
```

> 配合 `<svelte:boundary>` 可在父级追踪跨子组件的 promise 数量（边界隔离）。

## $effect.root()（独立作用域）

创建**非追踪**的作用域，不随组件销毁自动清理：

```js
import { flushSync } from 'svelte';

const destroy = $effect.root(() => {
  let count = $state(0);
  $effect(() => console.log('count =', count));
  return () => { /* cleanup */ };
});

count = 1;
flushSync(); // 立即跑 pending effects

// later...
destroy();
```

> 适用场景：测试工具、组件外创建 effect、模块级 effect。
> 文档："This rune also allows for the creation of effects outside of the component initialisation phase."

## When not to use $effect

| 需求 | 正确方案 |
|------|----------|
| 派生计算值 | `$derived` |
| 双向同步 | 函数绑定 `bind:value={() => v, setter}` |
| `await` 在模板 | 直接 `await`，或 `$derived.by(async)` |
| 单次初始化 | 模块顶层代码 / `onMount` 模式 |

### ❌ 用 $effect 同步派生

```svelte
<script>
  let count = $state(0);
  let doubled = $state();

  // ❌
  $effect(() => { doubled = count * 2; });

  // ✅
  let count = $state(0);
  let doubled = $derived(count * 2);
</script>
```

### ❌ 双向同步导致死循环

```svelte
<script>
  // ❌
  $effect(() => { left = total - spent; });
  $effect(() => { spent = total - left; });

  // ✅ 用 $derived + 函数绑定
  const total = 100;
  let spent = $state(0);
  let left = $derived(total - spent);
  function updateLeft(newLeft) { spent = total - newLeft; }
</script>

<label>
  <input type="range" bind:value={spent} max={total} />
  {spent}/{total} spent
</label>

<label>
  <input type="range" bind:value={() => left, updateLeft} max={total} />
  {left}/{total} left
</label>
```

### 实在要在 effect 中写 $state —— 用 untrack

```svelte
<script>
  import { untrack } from 'svelte';
  let count = $state(0);

  $effect(() => {
    const current = untrack(() => count);
    if (current < 10) count += 1; // 只读不追踪
  });
</script>
```

## 正确使用场景

- **Canvas / D3** 等命令式 API 同步
- **第三方库**订阅（CodeMirror、Mapbox 等）
- **事件监听**（window/document 事件 + cleanup）
- **网络请求**（带 AbortController 的 fetch）
- **分析埋点**（一次性副作用）

## SSR 行为

- `$effect` 在 SSR 时**不执行**
- 客户端 hydration 后才运行
- `$effect.root` 同理
- `$effect.pending` 在 SSR 时返回 0

## 性能提示

1. **避免在 effect 中读很多状态** → 把不相关的读包在 `untrack` 里
2. **避免无限循环** → 不要在 effect 中写依赖的状态
3. **debounce / throttle** 在 effect 内做（避免反复创建定时器）
4. **teardown 必须**清理副作用（订阅/定时器/事件监听）
5. **大计算用 `$derived` 缓存**而非每次 effect 跑
