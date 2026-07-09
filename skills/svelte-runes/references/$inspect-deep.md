# $inspect 完整参考

## 概述

`$inspect` 是开发时调试工具，**生产构建自动变为 noop**。

> 文档："The `$inspect` rune is roughly equivalent to `console.log`, with the exception that it will re-run whenever its argument changes. `$inspect` tracks reactive state deeply, meaning that updating something inside an object or array using fine-grained reactivity will cause it to re-fire."

## 基本语法

```svelte
<script>
  let count = $state(0);
  let message = $state('hello');

  // count 或 message 变化时打印
  $inspect(count, message);
</script>
```

- 跟踪**深度响应**：对象内部属性变化也触发
- 每次触发都会打印完整 stack trace（便于溯源）
- 第一次执行是 `init`，后续为 `update`

## $inspect(...).with()

自定义处理函数：

```svelte
<script>
  let count = $state(0);

  $inspect(count).with((type, count) => {
    if (type === 'update') {
      debugger;
    }
  });
</script>
```

签名：
```ts
$inspect(...values).with((type: 'init' | 'update', ...values) => void)
```

- 第一个参数是 `'init'` 或 `'update'`
- 后续参数是 `$inspect` 传入的值的**当前快照**

> 文档："The first argument to the callback is either `init` or `update`; subsequent arguments are the values passed to `$inspect`."

### 典型用法

```svelte
<script>
  $inspect(form).with((type, value) => {
    if (type === 'update') {
      console.log('form changed:', value);
      console.trace();
    }
  });
</script>
```

## $inspect.trace(...)（Svelte 5.14+）

让所在函数（effect / derived）被追踪：重跑时打印触发的状态：

```svelte
<script>
  $effect(() => {
    // 必须是函数体的第一条语句
    $inspect.trace();
    doSomeWork();
  });
</script>
```

可选 label：

```svelte
<script>
  $effect(() => {
    $inspect.trace('resize-handler');
    layout();
  });
</script>
```

> 文档："This rune, added in 5.14, causes the surrounding function to be traced in development. Any time the function re-runs as part of an effect or a derived, information will be printed to the console about which pieces of reactive state caused the effect to fire."

## 用在哪

- `$effect(...)` 函数体内
- `$derived.by(...)` 函数体内
- `$effect.pre(...)` 函数体内
- `$effect.root(...)` 内嵌的 effect

## 限制

- **不能用作业务逻辑** —— 是开发工具
- **生产构建**编译为 noop（无副作用）
- 第一次执行是 `init` 而不是 `update`
- `$inspect.trace` 必须是函数体的第一条语句

## 与 console.log 的区别

| | `console.log` | `$inspect` |
|---|---|---|
| 时机 | 调用时 | 状态变化时 |
| 深度追踪 | 手动 | 自动（响应式） |
| 触发栈 | 无 | 完整 stack |
| 多次值 | 一次性 | 持续追踪 |
| 生产环境 | 保留 | noop |

## 与 $effect 的关系

| | `$inspect` | `$effect` |
|---|---|---|
| 触发时机 | 状态变化后**立即**（push 通知） | microtask |
| 深度追踪 | 总是 | 只追踪同步读取 |
| 生产环境 | noop | 保留 |
| 用途 | 调试 | 副作用 |

## 典型模式

### 1. 调试对象变化

```svelte
<script>
  let user = $state({ name: 'Ada', age: 30 });
  $inspect(user); // 任意属性变化
</script>
```

### 2. 调试 effect 依赖

```svelte
<script>
  let a = $state(0);
  let b = $state(0);

  $effect(() => {
    $inspect.trace('compute');
    console.log(a + b);
  });
</script>
```

### 3. 在断点条件中使用

```svelte
<script>
  let count = $state(0);

  $inspect(count).with((type, value) => {
    if (type === 'update' && value === 10) {
      debugger; // count 变成 10 时中断
    }
  });
</script>
```

### 4. 调试回调/事件

```svelte
<script>
  function onClick() {
    $inspect(state); // 临时调试点击时的状态
  }
</script>
```

### 5. 调试跨模块状态

```svelte
<script>
  import { counter } from './state.svelte.js';
  $inspect(counter); // 任何使用方修改都看到
</script>
```

## 注意事项

1. **不进入模板** —— 在模板里 `{$inspect(x)}` 不会按预期工作
2. **不要嵌套 `$inspect`** 在大循环里 —— 性能开销
3. **不要依赖输出格式** —— 是给人看的，结构可能变
4. **配合 Svelte DevTools** 浏览器扩展使用更佳

## 与浏览器 DevTools 的协作

- `$inspect` 打印的值在 console 中可点击展开
- `debugger` 语句会触发断点（DevTools 打开时）
- `console.trace` 会打印调用栈
- 配合 `chrome://extensions` 中的 "Svelte DevTools" 可看组件树与每个组件状态

## SSR 行为

- 服务端渲染时 `$inspect` 不打印（不执行）
- 客户端 hydration 后开始工作
- `$inspect.trace` 同理
- `$inspect(...).with` 在 hydration 完成后才有意义

## 性能

- 每次状态变化触发回调 → 调试时避免放在 hot path
- 复杂计算不要用 `$inspect` 触发业务逻辑
- 生产构建无开销（编译移除）
