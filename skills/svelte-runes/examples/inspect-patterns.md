# $inspect Patterns

> `$inspect` 是开发时调试工具。生产构建中自动变为 noop。

## 1. 基础 —— 自动 console.log

```svelte
<script>
  let count = $state(0);
  let message = $state('hello');

  // count 或 message 变化时打印
  $inspect(count, message);
</script>

<button onclick={() => count++}>Increment</button>
<input bind:value={message} />
```

> 触发时**自动**打印当前值 + 调用栈（playground 除外）。

## 2. 多状态追踪

```svelte
<script>
  let count = $state(0);
  let user = $state({ name: 'Ada' });
  let list = $state([1, 2, 3]);

  // 任意变化都打印（含深层）
  $inspect(count, user, list);
</script>
```

> 深层代理：修改 `user.name` 或 `list.push(4)` 都会触发。

## 3. $inspect(...).with() —— 自定义处理

```svelte
<script>
  let count = $state(0);

  $inspect(count).with((type, count) => {
    if (type === 'update') {
      debugger; // 或 console.trace, analytics, etc.
    }
  });
</script>

<button onclick={() => count++}>Increment</button>
```

> `type` 是 `'init'` 或 `'update'`；后续参数是 `$inspect` 传入的值。

## 4. 调试用：把变化发到 server

```svelte
<script>
  let form = $state({ name: '', email: '' });

  $inspect(form).with((type, value) => {
    if (type === 'update') {
      fetch('/api/debug', {
        method: 'POST',
        body: JSON.stringify(value)
      });
    }
  });
</script>
```

## 5. $inspect.trace() —— 追踪 effect 依赖

```svelte
<script>
  import { doSomeWork } from './elsewhere';

  $effect(() => {
    // ⚠️ 必须是函数体的第一条语句
    $inspect.trace();
    doSomeWork();
  });
</script>
```

> 当 effect 重跑时，console 打印"哪些状态导致这次触发"。Svelte 5.14+ 引入。

## 6. $inspect.trace 加 label

```svelte
<script>
  $effect(() => {
    $inspect.trace('onResize');  // 自定义 label
    layout();
  });
</script>
```

## 7. $inspect.trace 配合 $derived

```svelte
<script>
  let count = $state(0);

  let doubled = $derived.by(() => {
    $inspect.trace(); // 看是哪些状态触发了这个 derived 重算
    return count * 2;
  });
</script>
```

## 8. 调试 mutation 路径

```svelte
<script>
  let user = $state({ name: 'Ada', age: 30 });

  $inspect(user).with((type, u) => {
    if (type === 'update') {
      console.trace('user changed');
    }
  });

  function rename(n) { user.name = n; }
</script>
```

## 9. 嵌套对象追踪

```svelte
<script>
  let config = $state({
    theme: { color: 'red' },
    layout: { width: 800 }
  });

  $inspect(config); // 任意深层变化都触发
</script>

<button onclick={() => (config.theme.color = 'blue')}>Change theme</button>
```

## 10. 与 SSR 的关系

```svelte
<!-- server-side: $inspect 不执行 -->
<!-- client-side: 在 hydration 后开始追踪 -->
```

> 服务端渲染时 `$inspect` 不打印；hydration 完成后才生效。

## 11. 性能 & 注意

- `$inspect` 不会在生产构建中保留（编译时移除）
- 每次状态变化都会触发回调 → 调试时避免放在 hot path
- 不要用 `$inspect` 触发**业务逻辑** —— 它是开发工具

## 12. $inspect 与 $effect 对比

| | `$inspect` | `$effect` |
|---|---|---|
| 时机 | 状态变化后立即（push 通知） | microtask |
| 深度追踪 | 总是 | 只追踪同步读取 |
| 产物 | 生产 noop | 永久保留 |
| 用途 | 调试 | 副作用 |

## 13. 完整调试组件示例

```svelte
<!-- DebugCounter.svelte -->
<script lang="ts">
  let count = $state(0);
  let lastTrigger: string | null = null;

  $inspect(count).with((type, value) => {
    lastTrigger = type;
    console.log(`[${type}] count =`, value);
  });

  $effect(() => {
    $inspect.trace('counter-side-effects');
    document.title = `count: ${count}`;
  });
</script>

<button onclick={() => count++}>{count}</button>
<p>Last trigger: {lastTrigger}</p>
```

## 14. 配合 Svelte DevTools

浏览器扩展 "Svelte DevTools" 会展示 `$inspect` 输出与每个组件的状态，比 `$inspect` 更强。但在没有扩展时 `$inspect` 是最直接的工具。
