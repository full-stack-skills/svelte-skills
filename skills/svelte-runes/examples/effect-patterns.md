# $effect Patterns

## 1. 基础用法

```svelte
<script>
  let count = $state(0);
  let input = $state('');

  // 追踪 count 和 input 变化
  $effect(() => {
    console.log(`count changed to: ${count}`);
    document.title = `${count} - ${input}`;
  });
</script>
```

Effect 在**挂载后**和**状态变化后（microtask）**执行。多次同步修改只触发一次重跑。

## 2. $effect + Canvas/DOM

```svelte
<script>
  let size = $state(100);
  let color = $state('#ff3e00');
  let canvas: HTMLCanvasElement;

  $effect(() => {
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = color;
    ctx.fillRect(0, 0, size, size);
  });
</script>

<canvas bind:this={canvas} width={200} height={200} />
<button onclick={() => size += 10}>Grow</button>
<input bind:value={color} type="color" />
```

## 3. $effect cleanup（定时器/订阅）

```svelte
<script>
  let seconds = $state(0);
  let running = $state(true);

  $effect(() => {
    if (!running) return;
    const id = setInterval(() => seconds += 1, 1000);
    return () => clearInterval(id); // cleanup
  });
</script>

<p>{seconds}s</p>
<button onclick={() => running = !running}>Pause/Resume</button>
```

teardown 在 (a) 父 effect 重跑前、(b) 父级销毁时执行。

## 4. $effect.pre（DOM 更新前）

```svelte
<script>
  import { tick } from 'svelte';

  let messages = $state<string[]>([]);
  let div: HTMLDivElement;

  // 在 DOM 更新前执行（如计算 scrollHeight）
  $effect.pre(() => {
    if (!div) return;
    messages.length; // 显式依赖追踪
    if (div.offsetHeight + div.scrollTop > div.scrollHeight - 20) {
      tick().then(() => {
        div.scrollTo(0, div.scrollHeight);
      });
    }
  });
</script>

<div bind:this={div}>
  {#each messages as msg}
    <p>{msg}</p>
  {/each}
</div>
```

除时序外，`$effect.pre` 与 `$effect` 完全一致。

## 5. $effect.tracking（判断追踪上下文）

```svelte
<script>
  console.log('setup:', $effect.tracking()); // false

  $effect(() => {
    console.log('effect:', $effect.tracking()); // true
    if (!$effect.tracking()) {
      // 非追踪上下文（如事件处理器）
    }
  });
</script>

<p>tracking in template: {$effect.tracking()}</p>
```

> 用于实现 `createSubscriber` 这类只在被追踪时才订阅的工具。

## 6. $effect + setTimeout（不追踪）

```svelte
<script>
  let count = $state(0);

  // ⚠️ setTimeout 内部不追踪 count
  $effect(() => {
    setTimeout(() => {
      console.log(count); // 闭包捕获的值，不追踪
    }, 1000);
  });

  // ✅ 想要追踪，把状态写到 $state
  let savedCount = $state(0);
  $effect(() => {
    savedCount = count; // 追踪
    setTimeout(() => {
      console.log(savedCount);
    }, 1000);
  });
</script>
```

## 7. $effect.pending()（待定 Promise 数量）

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  async function add(a, b) {
    await new Promise((f) => setTimeout(f, 500));
    return a + b;
  }
</script>

<button onclick={() => a++}>a++</button>
<button onclick={() => b++}>b++</button>

<p>{a} + {b} = {await add(a, b)}</p>

{#if $effect.pending()}
  <p>pending promises: {$effect.pending()}</p>
{/if}
```

> 返回**当前 boundary**（不含子 boundary）中待定的 Promise 数。可用于全局 loading 状态。

## 8. $effect.root()（独立作用域，手动清理）

```js
// setup-test.js
import { flushSync } from 'svelte';

const destroy = $effect.root(() => {
  let count = $state(0);
  $effect(() => console.log('count =', count));
  return () => { /* cleanup */ };
});

count = 1;
flushSync(); // 立即执行 pending effects

destroy();
```

> 用于测试工具、组件外手动控制作用域。`$effect.root` 创建的 effect **不会随组件销毁自动清理** —— 必须调用返回的 `destroy()`。

## 9. ❌ 错误：不要用 $effect 同步派生值

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  // ❌ 错误做法
  let sum;
  $effect(() => { sum = a + b; });

  // ✅ 正确做法
  let sum = $derived(a + b);

  // ❌ 错误：双向同步 → 无限循环
  let left = $state(100);
  let right = $state(100);
  $effect(() => { left = 100 - right; });
  $effect(() => { right = 100 - left; });
</script>
```

## 10. 正确：函数绑定代替双向 $effect

```svelte
<script>
  const total = 100;
  let spent = $state(0);
  let left = $derived(total - spent);

  function updateLeft(newLeft) { spent = total - newLeft; }
</script>

<label>
  <input type="range" bind:value={spent} max={total} />
  spent: {spent}
</label>

<label>
  <input
    type="range"
    bind:value={() => left, updateLeft}
    max={total}
  />
  left: {left}
</label>
```

## 11. $effect + untrack（打破追踪）

```svelte
<script>
  import { untrack } from 'svelte';

  let count = $state(0);
  let multiplier = $state(2);

  $effect(() => {
    // 追踪 count，但不追踪 multiplier
    const m = untrack(() => multiplier);
    console.log(`${count} × ${m} = ${count * m}`);
  });
</script>
```

> 用 untrack 来读"无关"状态，避免误触发重跑。

## 12. 条件分支决定依赖

effect **只追踪上次运行时同步读过的状态**：

```svelte
<script>
  import confetti from 'canvas-confetti';

  let condition = $state(true);
  let color = $state('#ff3e00');

  $effect(() => {
    if (condition) {
      confetti({ colors: [color] });   // condition/color 都是依赖
    } else {
      confetti();                        // 只追踪 condition
    }
  });
</script>
```

切换 `condition` 不会立即读 `color` —— 直到 `condition` 再次为真。

## 13. 订阅第三方库

```svelte
<script>
  import { onMount } from 'svelte';
  let count = $state(0);

  $effect(() => {
    const handler = (e) => count += e.delta;
    window.addEventListener('wheel', handler, { passive: true });
    return () => window.removeEventListener('wheel', handler);
  });
</script>
```

## 14. fetch / 网络请求副作用

```svelte
<script>
  let query = $state('');
  let data = $state(null);
  let error = $state(null);

  $effect(() => {
    // 每次 query 变化取消上一次请求
    const ctrl = new AbortController();
    fetch(`/api/search?q=${query}`, { signal: ctrl.signal })
      .then(r => r.json())
      .then(d => { data = d; error = null; })
      .catch(e => { if (e.name !== 'AbortError') error = e; });
    return () => ctrl.abort();
  });
</script>
```

## 15. 何时不用 $effect

| 需求 | 正确方案 |
|------|----------|
| 派生计算值 | `$derived` |
| 双向同步 | 函数绑定 `bind:value={() => v, setter}` |
| 表达式中含 await | 模板直接 `await` 或 `$derived.by(async)` |
| 无限循环 | 用 `untrack` 包裹读取，或改用 `$derived` |

## 16. 在类中谨慎使用 $effect

类字段里调用 `$effect` 时，effect 函数体内读取的状态**不会**作为依赖：

```svelte
<script>
  class AutoLogger {
    value = $state(0);
    log = $effect(() => {
      // 这里读 value 不会建立依赖
      console.log('initial:', this.value);
    });
  }
</script>
```

> 文档原文："If `$state` and `$derived` are used directly inside the `$effect` (for example, during creation of a reactive class), those values will not be treated as dependencies." 因此推荐把 effect 放在组件而非类里。
