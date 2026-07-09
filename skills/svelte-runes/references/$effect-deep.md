# $effect 深层机制

## 基本执行时机

```svelte
<script>
  let count = $state(0);

  $effect(() => {
    // 1. 组件挂载后首次执行
    // 2. count 变化后重执行
    console.log('effect ran, count =', count);
  });
</script>
```

## cleanup

```svelte
<script>
  let enabled = $state(true);
  let tick = $state(0);

  $effect(() => {
    if (!enabled) return;

    const id = setInterval(() => tick += 1, 1000);

    // 返回 cleanup 函数
    return () => clearInterval(id);
  });
</script>
```

## $effect.pre（DOM 更新前）

```svelte
<script>
  let count = $state(0);

  // DOM 更新前执行，可用于测量/修改
  $effect.pre(() => {
    console.log('before DOM update, count =', count);
  });
</script>
```

## $effect.tracking（判断上下文）

```svelte
<script>
  let count = $state(0);

  // 在 reactive 上下文中返回 true
  $effect(() => {
    if (!$effect.tracking()) {
      // 非追踪上下文（如事件处理器、setTimeout）
    }
  });
</script>
```

## $effect.root（测试用）

```svelte
<script>
  import { flushSync } from 'svelte';

  // 在测试中包装 effect
  const cleanup = $effect.root(() => {
    let count = $state(0);
    $effect(() => console.log(count));
    return () => { /* cleanup */ };
  });

  count = 1;
  flushSync(); // 立即执行所有 pending effects

  cleanup();
</script>
```

## 常见错误

### 1. 用 $effect 同步派生值
```svelte
// ❌ 错误
let doubled;
$effect(() => { doubled = count * 2; });

// ✅ 正确
let doubled = $derived(count * 2);
```

### 2. 双向同步导致死循环
```svelte
// ❌ 死循环
$effect(() => { a = b + 1; });
$effect(() => { b = a + 1; });

// ✅ 正确：单向派生
let a = $state(0);
let b = $derived(a + 1);
```

### 3. 异步闭包不追踪
```svelte
$effect(() => {
  setTimeout(() => console.log(count), 100);
  // count 变化不会重跑 effect
});

// ✅ 如需追踪
let saved = $state(0);
$effect(() => { saved = count; });
$effect(() => {
  setTimeout(() => console.log(saved), 100);
});
```

## 正确使用场景

✅ **外部 API 同步**（如 D3、canvas）：
```svelte
$effect(() => {
  const ctx = canvas.getContext('2d');
  ctx.clearRect(0, 0, width, height);
  drawChart(data);
});
```

✅ **订阅外部事件**：
```svelte
$effect(() => {
  const handler = (e) => count += e.delta;
  window.addEventListener('wheel', handler);
  return () => window.removeEventListener('wheel', handler);
});
```
