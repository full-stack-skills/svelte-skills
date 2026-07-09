---
name: svelte-runes
description: Svelte 5 Runes 响应式系统技能。当用户需要使用 $state/$derived/$effect/$props/$bindable/$inspect 等符文，或理解 Svelte 5 显式响应式与 Svelte 4 隐式响应式的区别时使用。
---

# Svelte Runes Reference (Svelte 5)

本技能覆盖 Svelte 5 的 Runes（符文）系统。Runes 是 Svelte 5 引入的显式响应式语法，取代了 Svelte 4 的隐式 `let` 声明和 `$:` 语句。

## When to use this skill

当用户需要理解或使用 `$state`、`$derived`、`$effect`、`$props`、`$bindable`、`$inspect` 等符文，或需要将 Svelte 4 代码迁移到 Svelte 5 时使用本技能。

## Critical: Runes Overview

Runes 是以 `$` 为前缀的符号，类似于函数调用语法：

```js
let message = $state('hello');
```

关键区别：
- **无需导入** — Runes 是语言内置关键字
- **不是值** — 不能赋值给变量或作为函数参数
- **位置敏感** — 仅在特定位置有效（编译器会报错）

| Rune | 用途 |
|------|------|
| `$state` | 创建响应式状态 |
| `$derived` | 声明派生计算值 |
| `$effect` | 声明副作用 |
| `$props` | 声明组件属性 |
| `$bindable` | 可绑定 prop |
| `$inspect` | 开发调试 |
| `$host` | 自定义元素访问 |

## Critical: $state

创建响应式状态，UI 在状态变化时自动更新。

```svelte
let count = $state(0);
let user = $state({ name: 'Ada', age: 30 });
```

### 深层响应式代理

`$state` 对数组和简单对象自动创建深层 Proxy，属性变更自动触发更新：

```js
let todos = $state([
  { done: false, text: 'task' }
]);

todos[0].done = true;          // ✅ 触发更新
todos.push({ done: false });  // ✅ 触发更新
```

### $state.raw（避免深层代理）

适用于大数组和无需深层变更的场景：

```js
let list = $state.raw([]);

// 只能重新赋值，不能 .push()
list = [...list, newItem]; // ✅
list.push(newItem);         // ❌ 无效
```

### $state.snapshot（静态快照）

获取深层代理的只读快照（用于传给外部 API）：

```svelte
console.log($state.snapshot(proxyObject));
```

### $state.eager（即时 UI 更新）

用于 `await` 表达式中立即更新 UI：

```svelte
<nav>
  <a href="/" aria-current={$state.eager(pathname) === '/' ? 'page' : null}>home</a>
</nav>
```

### 类字段中的 $state

```js
class Counter {
  count = $state(0);          // 公共字段
  #value = $state(0);          // 私有字段

  constructor(start = 0) {
    this.value = $state(start); // constructor 中初始化
  }
}
```

> 编译器将 `$state` 字段转换为 `get`/`set` 方法，指向私有字段。

### 解构陷阱

解构后丢失响应式（与普通 JS 行为一致）：

```js
let { name, age } = $state({ name: 'Ada', age: 30 });
name = 'Bob'; // ❌ 不会触发更新，原对象不变
```

### 跨模块传递状态

`.svelte.js/.svelte.ts` 文件中可使用 Runes，但不能直接 `export let` 重新赋值的 `$state`：

```js
// ❌ 不可行
export let count = $state(0);

// ✅ 方案1：不重新赋值整个对象
export const counter = $state({ count: 0 });
export function increment() { counter.count += 1; }

// ✅ 方案2：用 getter/setter 封装
let _count = $state(0);
export function getCount() { return _count; }
export function setCount(n) { _count = n; }
```

## Critical: $derived

声明派生值——基于已有状态的只读计算值。

```svelte
let count = $state(0);
let doubled = $derived(count * 2);
```

### $derived.by（复杂派生）

```svelte
let total = $derived.by(() => {
  let sum = 0;
  for (const item of items) sum += item.price;
  return sum;
});
```

### 派生值覆盖（Svelte 5.25+，乐观 UI）

```svelte
let likes = $derived(post.likes);

async function onclick() {
  likes += 1; // 即时乐观更新
  try {
    await like();
  } catch {
    likes -= 1; // 回滚
  }
}
```

### 理解依赖追踪

`$derived` 内部同步读取的所有 `$state`/`$derived` 都是依赖：

```js
let a = Promise.resolve(1);
let b = 2;
let sum = $derived(await a + b);
// a 和 b 都是依赖（await 之后的同步读取也会追踪）
```

用 `untrack` 排除非依赖值。

### 浅层 vs 深层

`$state` 对象/数组是深层代理；`$derived` 值保持原样：

```js
let items = $state([...]);
let selected = $derived(items[0]);
selected.name = 'new'; // ✅ 影响底层 items
```

## Critical: $effect

声明副作用——DOM 操作、第三方库调用、网络请求等。**不应**用 `$effect` 同步状态。

```svelte
$effect(() => {
  document.title = `count: ${count}`;
  return () => { /* 清理函数 */ };
});
```

### 依赖追踪

自动追踪 `$state`/`$derived` 的同步读取：

```svelte
$effect(() => {
  // color 和 size 是依赖
  ctx.fillStyle = color;
  ctx.fillRect(0, 0, size, size);
});
```

### 异步不追踪

`await` 之后和 `setTimeout` 内部的读取**不**追踪：

```js
$effect(() => {
  ctx.fillStyle = color;   // ✅ 追踪
  setTimeout(() => {
    ctx.fillRect(0, 0, size, size); // ❌ size 不追踪
  }, 0);
});
```

### $effect.pre（DOM 更新前运行）

```svelte
$effect.pre(() => {
  if (!div) return; // 挂载前跳过
  messages.length;  // 显式依赖追踪
  // DOM 更新前执行（如滚动位置计算）
});
```

### $effect.tracking（判断追踪上下文）

```svelte
console.log($effect.tracking()); // false（组件初始化）

$effect(() => {
  console.log($effect.tracking()); // true（effect 内）
});
```

### 清理函数（Teardown）

```svelte
$effect(() => {
  const interval = setInterval(() => count += 1, 1000);
  return () => clearInterval(interval); // 清理
});
```

### 何时不用 $effect

| 需求 | 正确方案 |
|------|----------|
| 派生计算值 | `$derived` |
| 双向同步 | 函数绑定 `bind:value={() => v, setter}` |
| 无限循环 | `untrack` 包裹读取 |

## Critical: $props

声明组件属性（props）：

```svelte
let { name, age = 18, ...rest } = $props();
```

### 类型安全

```svelte
<script lang="ts">
  let { title }: { title: string } = $props();
</script>
```

### prop 默认值

```svelte
let { adjective = 'happy', count = 0 } = $props();
```

### prop 重命名

```svelte
let { super: hero = 'default' } = $props();
```

### 更新 props

Props 在父组件变化时自动更新，但子组件**不应直接修改** prop（除 `$bindable` 外）：

```svelte
// ❌ 不要修改
let { object } = $props();
object.count += 1; // 警告：ownership_invalid_mutation

// ✅ 用 callback props 或 $bindable
```

## Critical: $bindable

允许子组件修改父组件状态的 prop 类型：

```svelte
// FancyInput.svelte
let { value = $bindable(), ...props } = $props();
<input bind:value />

// 父组件
let message = $state('hello');
<FancyInput bind:value={message} />
```

## Critical: $inspect / $inspect.trace

开发时打印状态变化（生产环境为 noop）：

```svelte
$inspect(count, name); // 变化时自动打印

// 自定义处理器
$inspect(count).with((type, val) => {
  if (type === 'update') debugger;
});

// 追踪依赖
$effect(() => {
  $inspect.trace();
  doSomeWork();
});
```

## Critical: $host

仅在编译为自定义元素时使用，访问宿主元素：

```svelte
<svelte:options customElement="my-stepper" />

<script>
  function dispatch(type) {
    $host().dispatchEvent(new CustomEvent(type));
  }
</script>

<button onclick={() => dispatch('increment')}>+</button>
```

## Quick Fixes

| 问题 | 解决方案 |
|------|----------|
| 状态变化不更新 UI | 确认用了 `$state`（不是普通 `let`） |
| `$derived` 不生效 | 依赖必须同步读取（不在 `await` 后） |
| `$effect` 无限循环 | 不要在其中直接修改 `$state`，改用 `$derived` |
| 类方法中 `this` 丢失 | 用箭头函数字段或内联函数 |
| prop 变异警告 | 用 `$bindable` 或 callback props |
| 解构后响应式丢失 | 访问原对象属性而非解构变量 |

## Gotchas

1. **`$state` 是深层代理** — 解构后丢失响应式，访问原始对象属性
2. **`$effect` 不追踪异步读取** — `await`/`setTimeout` 后的读取不在依赖中
3. **类中 `$effect` 字段** — 方法内读取的状态不作为依赖追踪
4. **`$effect` 不应在 SSR 运行** — 浏览器专用，SSR 时自动跳过
5. **`$props` 默认值不是代理** — 非 `$bindable` 的 prop fallback 值不是响应式对象

## FAQ

**Q: `$derived` 和 `$state` 的区别？**
A: `$state` 创建可变状态；`$derived` 创建只读派生值，自动从依赖推导。

**Q: 什么时候用 `$effect`？**
A: 仅用于副作用：DOM 操作、第三方库调用、网络请求。派生值同步状态永远不用 `$effect`。

**Q: `$state.raw` 和普通 `$state` 的区别？**
A: `$state.raw` 不对数组/对象创建深层代理，性能更好，但只能通过重新赋值来更新。

**Q: `$props` 能解构吗？**
A: 可以，且支持默认值、重命名、rest 解构。

## Examples

可执行的代码示例，见 `examples/` 目录：

| 文件 | 内容 |
|------|------|
| `state-patterns.md` | `$state` 基础、深层代理、类字段、跨模块共享 |
| `derived-patterns.md` | `$derived` 基础、$derived.by、乐观 UI、解构派生 |
| `effect-patterns.md` | `$effect` 基础、cleanup、pre、追踪规则、禁忌 |
| `props-patterns.md` | `$props` 解构、$bindable、Rest Props、泛型组件 |

## References

深入技术参考，见 `references/` 目录：

| 文件 | 内容 |
|------|------|
| `runes-overview.md` | Runes 整体架构、vs Legacy 对比、生命周期 |
| `$state-deep.md` | Proxy 行为、$state.raw/snapshot/eager、类中 $state |
| `$derived-deep.md` | 表达式 vs $derived.by、依赖追踪、可写派生 |
| `$effect-deep.md` | pre/cleanup/root、追踪规则、常见错误 |
| `$props-deep.md` | 解构语法、$bindable、Rest Props、泛型 |
| `context-deep.md` | createContext vs setContext、类型安全、SSR |
