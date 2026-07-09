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

## 4. $effect.pre（DOM 更新前）

```svelte
<script>
  let messages = $state<string[]>([]);
  let container: HTMLDivElement;

  // 在 DOM 更新前执行（如计算 scrollHeight）
  $effect.pre(() => {
    if (!container) return;
    messages.length; // 显式依赖
    if (container.scrollTop + container.clientHeight >= container.scrollHeight - 50) {
      // 用户已滚动到底部
    }
  });
</script>

<div bind:this={container}>
  {#each messages as msg}
    <p>{msg}</p>
  {/each}
</div>
```

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

<p>tracking: {$effect.tracking()}</p>
```

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

  // ✅ 如需追踪，在 effect 外处理
  let savedCount = $state(0);
  $effect(() => {
    savedCount = count; // 追踪
    setTimeout(() => {
      console.log(savedCount); // 用 savedCount
    }, 1000);
  });
</script>
```

## 7. ❌ 错误：不要用 $effect 同步派生值

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  // ❌ 错误做法
  let sum;
  $effect(() => { sum = a + b; });

  // ✅ 正确做法
  let sum = $derived(a + b);

  // ❌ 错误：双向同步
  let left = $state(100);
  let right = $state(100);
  $effect(() => { left = 100 - right; }); // 无限循环！
  $effect(() => { right = 100 - left; }); // 无限循环！
</script>
```

## 8. 正确：函数绑定代替双向 $effect

```svelte
<script>
  let total = 100;
  let spent = $state(0);
  let left = $derived(total - spent);

  // ✅ 用函数绑定做双向同步
  function updateLeft(newLeft) { spent = total - newLeft; }
</script>

<label>
  <input type="range" bind:value={spent} max={total} />
  spent: {spent}
</label>

<label>
  <!-- 用函数绑定更新 left -->
  <input
    type="range"
    bind:value={() => left, updateLeft}
    max={total}
  />
  left: {left}
</label>
```

## 9. $effect + untrack（打破追踪）

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
