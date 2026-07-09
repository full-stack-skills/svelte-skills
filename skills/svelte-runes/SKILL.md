---
name: svelte-runes
description: Svelte 5 Runes 响应式系统技能。当用户需要使用 $state/$derived/$effect/$props/$bindable/$inspect/$host 等符文，或理解 Svelte 5 显式响应式与 Svelte 4 隐式响应式的区别时使用。
---

# Svelte Runes Reference (Svelte 5)

本技能覆盖 Svelte 5 的 Runes（符文）系统。Runes 是 Svelte 5 引入的显式响应式语法，取代了 Svelte 4 的隐式 `let` 声明和 `$:` 语句。

## When to use this skill

当用户需要理解或使用 `$state`、`$derived`、`$effect`、`$props`、`$bindable`、`$inspect`、`$host` 等符文，或需要将 Svelte 4 代码迁移到 Svelte 5 时使用本技能。

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

> `$state.raw` 内部仍可包含响应式状态（如原始数组里放代理对象）。

### $state.snapshot（静态快照）

获取深层代理的只读快照（用于传给外部 API）：

```svelte
console.log($state.snapshot(proxyObject));
```

> 若值有 `toJSON()` 方法，snapshot 会克隆 `toJSON()` 的返回值。

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

  // 箭头函数字段 → this 自动绑定
  reset = () => { this.count = 0; };
}
```

> 编译器将 `$state` 字段转换为原型上的 `get`/`set` 方法，指向私有字段。**这些属性不可枚举**。注意 `this` 绑定：方法直接传给事件处理器会丢失 `this`，用箭头函数字段或内联 `() => todo.reset()`。

### 内置响应式类（svelte/reactivity）

普通 `Map`/`Set`/`Date`/`URL` 不响应 —— 改用响应式版本：

```js
import { SvelteSet, SvelteMap, SvelteDate, SvelteURL } from 'svelte/reactivity';

const tags = new SvelteSet(['a', 'b']);
const cache = new SvelteMap();
tags.add('c');      // ✅ 触发更新
cache.set('a', 1);  // ✅ 触发更新
```

### 解构陷阱

解构后丢失响应式（与普通 JS 行为一致）：

```js
let { name, age } = $state({ name: 'Ada', age: 30 });
name = 'Bob'; // ❌ 不会触发更新，原对象不变
```

### 跨模块传递状态

`.svelte.js/.svelte.ts` 文件中可使用 Runes，但不能直接 `export let` 重新赋值的 `$state`：

```js
// ❌ 不可行：另一文件读到的会是 Signal 对象
export let count = $state(0);

// ✅ 方案1：不重新赋值整个对象
export const counter = $state({ count: 0 });
export function increment() { counter.count += 1; }

// ✅ 方案2：模块内私有 + 函数导出
let _count = $state(0);
export function getCount() { return _count; }
export function setCount(n) { _count = n; }
```

### 将 state 传入函数 —— getter 模式

JavaScript 是**按值传递**，`$state` 同理。若函数需读取最新值，传入 getter：

```js
/** @param {() => number} getA @param {() => number} getB */
function add(getA, getB) {
  return () => getA() + getB();
}

let a = $state(1);
let b = $state(2);
const total = add(() => a, () => b);
console.log(total()); // 3
a = 3; b = 4;
console.log(total()); // 7
```

> 也可借助 proxy 属性或 get/set 属性实现"实时读取"。文档："Note that 'functions' is broad — it encompasses properties of proxies and get/set properties."

## Critical: $derived

声明派生值——基于已有状态的只读计算值。

```svelte
let count = $state(0);
let doubled = $derived(count * 2);
```

> 表达式**必须纯净** —— 内部不能修改 `$state`（编译器报错）。

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

> `const` 派生只读；用 `let` 才能重新赋值。文档："Prior to Svelte 5.25, deriveds were read-only."

### 理解依赖追踪

`$derived` 内部同步读取的所有 `$state`/`$derived` 都是依赖：

```js
let a = Promise.resolve(1);
let b = 2;
let sum = $derived(await a + b);
// a 和 b 都是依赖（await 之后的同步读取也会追踪）
```

> 仅**表达式自身**的 `await` 后的同步读才算依赖；**调用函数**内部的 `await` 不计入。

用 `untrack` 排除非依赖值。

### Push-pull 响应

派生值**只在被读取时**重新计算（pull），但状态变化时**立即通知**所有依赖（push）。若派生返回的引用未变，下游不更新：

```svelte
let count = $state(0);
let large = $derived(count > 10); // 布尔
// 大文本节点只在 large 变化时重渲染，而非 count
```

> 派生内避免 `() => ({})`、`[...].map(...)` 这类返回新引用的写法 —— 会让下游始终重算。

### 派生不解构代理

`$derived` 不会把返回值包成 Proxy —— 修改派生返回对象的属性会影响到底层 `$state`：

```js
let items = $state([...]);
let selected = $derived(items[0]);
selected.name = 'new'; // ✅ 影响 items[0].name
```

### 派生解构

```js
let { first, last } = $derived(user);
// 等价于：
// let first = $derived(user.first);
// let last = $derived(user.last);
```

## Critical: $effect

声明副作用——DOM 操作、第三方库调用、网络请求等。**不应**用 `$effect` 同步状态。

```svelte
$effect(() => {
  document.title = `count: ${count}`;
  return () => { /* 清理函数 */ };
});
```

### 生命周期

- 挂载后**首次执行**
- 状态变化时 **microtask 调度**（批量）
- DOM 更新**之后**执行
- 仅浏览器执行（SSR 自动跳过）
- 可在组件任意位置调用，只要在父 effect 运行期间

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

### 条件分支决定依赖

effect **只追踪上次运行时实际读取的状态**：

```ts
$effect(() => {
  if (condition) {
    confetti({ colors: [color] });
  } else {
    confetti();
  }
});
// condition=false 时 color 不再是依赖
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

> 用于 `createSubscriber` 这类工具：仅在状态被追踪时订阅，事件处理器中不订阅。

### $effect.pending()（待定 Promise 数量）

返回**当前 boundary**（不含子 boundary）中待定的 Promise 数量：

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
  <span>pending: {$effect.pending()}</span>
{/if}
```

### $effect.root()（独立作用域）

创建**非追踪**作用域，不随组件销毁自动清理：

```js
import { flushSync } from 'svelte';

const destroy = $effect.root(() => {
  let count = $state(0);
  $effect(() => console.log(count));
  return () => { /* cleanup */ };
});

count = 1;
flushSync(); // 立即跑 pending effects

// later...
destroy();
```

> 适用：测试工具、组件外创建 effect、模块级 effect。文档："This rune also allows for the creation of effects outside of the component initialisation phase."

### 清理函数（Teardown）

```svelte
$effect(() => {
  const interval = setInterval(() => count += 1, 1000);
  return () => clearInterval(interval); // 清理
});
```

> teardown 在 (a) effect 重跑前、(b) 组件销毁时执行。

### 何时不用 $effect

| 需求 | 正确方案 |
|------|----------|
| 派生计算值 | `$derived` |
| 双向同步 | 函数绑定 `bind:value={() => v, setter}` |
| 无限循环 | `untrack` 包裹读取 |
| 模板内 await | 直接 `await` |

> 文档："In general, `$effect` is best considered something of an escape hatch — useful for things like analytics and direct DOM manipulation — rather than a tool you should use frequently."

## Critical: $props

声明组件属性（props）：

```svelte
let { name, age = 18, ...rest } = $props();
```

### 类型安全

```svelte
<script lang="ts">
  // 方式 1：内联注解
  let { title }: { title: string } = $props();

  // 方式 2：接口分离
  interface Props { adjective: string; }
  let { adjective }: Props = $props();
</script>
```

JSDoc 风格：

```svelte
<script>
  /** @type {{ adjective: string }} */
  let { adjective } = $props();
</script>
```

> 原生 DOM 元素的属性类型在 `svelte/elements` 中（如 `HTMLButtonAttributes`）。

### prop 默认值

```svelte
let { adjective = 'happy', count = 0 } = $props();
```

> 默认值**不是**响应式代理 —— 父组件未传时修改其属性不触发更新。文档："Fallback values are not turned into reactive state proxies."

### prop 重命名

```svelte
let { super: hero = 'default' } = $props();
let { 'class': className, 'for': htmlFor } = $props();
```

### 更新 props

Props 在父组件变化时自动更新，但子组件**不应直接修改** prop（除 `$bindable` 外）：

```svelte
// ❌ 不要修改
let { object } = $props();
object.count += 1; // 警告：ownership_invalid_mutation

// ✅ 用 callback props 或 $bindable
```

子组件**可以临时重新赋值** prop（unsaved 状态），父级不受影响。

### Rest Props

```svelte
let { variant = 'primary', size = 'md', ...rest } = $props();

<button class="btn btn-{variant} btn-{size}" {...rest}>
  <slot />
</button>
```

> 类型：`{ variant?: 'primary' | 'secondary'; [key: string]: any }` 或 `HTMLButtonAttributes`。

### $props.id()（Svelte 5.20+）

生成**当前组件实例唯一**的 ID，SSR 时 server/client 一致：

```svelte
<script>
  const uid = $props.id();
</script>

<form>
  <label for="{uid}-firstname">First Name:</label>
  <input id="{uid}-firstname" type="text" />
</form>
```

> 适用：`for`/`aria-labelledby`/`aria-describedby` 等需要 ID 配对的场景。

### 泛型组件

```svelte
<script lang="ts" generics="T extends { id: number }">
  import type { Snippet } from 'svelte';

  let {
    items,
    renderItem
  }: {
    items: T[];
    renderItem: Snippet<[T]>;
  } = $props();
</script>

{#each items as item (item.id)}
  {@render renderItem(item)}
{/each}
```

## Critical: $bindable

允许子组件修改父组件状态的 prop 类型——双向数据流：

```svelte
<!-- FancyInput.svelte -->
<script>
  let { value = $bindable(), ...props } = $props();
</script>

<input bind:value={value} {...props} />
```

```svelte
<!-- 父组件 -->
<script>
  import FancyInput from './FancyInput.svelte';
  let message = $state('hello');
</script>

<FancyInput bind:value={message} />
<p>{message}</p>
```

### 用法

```js
// 必须放在解构的默认值位置
let {
  value = $bindable(),         // 必填
  other = $bindable('hello'),  // 带 fallback
} = $props();
```

### 父组件两种用法

```svelte
<!-- 双向 -->
<FancyInput bind:value={message} />

<!-- 单向（父不监听子） -->
<FancyInput value={message} />
```

> 文档："The parent component doesn't have to use `bind:` — it can just pass a normal prop. Some parents don't want to listen to what their children have to say."

### 何时使用

| 场景 | 推荐 |
|------|------|
| 受控表单组件 | `$bindable` |
| 子→父事件通知 | callback props |
| 跨组件共享 | Context / store |

> 文档原话："This isn't something you should do often — overuse can make your data flow unpredictable and your components harder to maintain."

### 与函数绑定组合

```svelte
<script>
  let { value = $bindable(0), min = 0, max = 100 } = $props();
  function clamp(v: number) { return Math.max(min, Math.min(max, v)); }
</script>

<input
  type="range"
  bind:value={() => value, (v) => (value = clamp(v))}
  {min} {max}
/>
```

## Critical: $inspect / $inspect.trace

开发时打印状态变化（**生产环境为 noop**）：

```svelte
<script>
  let count = $state(0);
  let message = $state('hello');

  $inspect(count, message); // 变化时自动 console.log
</script>
```

> 深度追踪 —— 修改 `user.name` 或 `arr.push(x)` 都触发。Stack trace 同步打印（playground 除外）。

### $inspect(...).with() —— 自定义处理

```svelte
$inspect(count).with((type, count) => {
  if (type === 'update') {
    debugger; // 或 console.trace、埋点等
  }
});
```

签名：`(type: 'init' | 'update', ...values) => void`。

### $inspect.trace()（Svelte 5.14+）

让所在函数被追踪，重跑时打印触发的状态：

```svelte
<script>
  $effect(() => {
    // ⚠️ 必须是函数体的第一条语句
    $inspect.trace();
    doSomeWork();
  });
</script>
```

可选 label：

```svelte
$effect(() => {
  $inspect.trace('resize-handler');
  layout();
});
```

> 文档："Any time the function re-runs as part of an effect or a derived, information will be printed to the console about which pieces of reactive state caused the effect to fire."

## Critical: $host

仅在编译为**自定义元素**时使用，访问宿主元素：

```svelte
<svelte:options customElement="my-stepper" />

<script>
  function dispatch(type) {
    $host().dispatchEvent(new CustomEvent(type));
  }
</script>

<button onclick={() => dispatch('decrement')}>-</button>
<button onclick={() => dispatch('increment')}>+</button>
```

```svelte
<!-- App.svelte -->
<script>
  import './Stepper.svelte';
  let count = $state(0);
</script>

<my-stepper
  ondecrement={() => (count -= 1)}
  onincrement={() => (count += 1)}
></my-stepper>
```

### 典型用例

- **派发自定义事件**：`$host().dispatchEvent(new CustomEvent('x', { detail, bubbles: true, composed: true }))`
- **访问宿主属性**：`$host().getAttribute('foo')` / `setAttribute(...)`
- **调用宿主方法**：`$host().focus()` 等
- **暴露 imperative API**：用 `export function`

### 自定义元素配置

```svelte
<svelte:options
  customElement="my-tag"
  customElementShadow="open"           <!-- 或 "closed" / "none" -->
  customElementEvents={['change']}     <!-- 声明可冒泡事件 -->
  customElementObservers={{ attr: ['data-foo'] }}  <!-- 观察的 attribute -->
/>
```

### 错误用法

```svelte
<!-- ❌ 普通组件（非 custom element）中调用 -->
<script>
  $host(); // 编译错误
</script>

<!-- ❌ 用 $host().querySelector 找内部元素 -->
<script>
  $host().querySelector('.child'); // 改用 bind:this
</script>
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
| Map/Set 修改不触发更新 | 改用 `SvelteMap` / `SvelteSet` |
| 跨文件 `$state` 读到 Signal 对象 | 用不可重新赋值的对象 + 导出函数 |

## Gotchas

1. **`$state` 是深层代理** — 解构后丢失响应式，访问原始对象属性
2. **`$effect` 不追踪异步读取** — `await`/`setTimeout` 后的读取不在依赖中
3. **类中 `$effect` 字段** — 方法内读取的状态不作为依赖追踪
4. **`$effect` 不应在 SSR 运行** — 浏览器专用，SSR 时自动跳过
5. **`$props` 默认值不是代理** — 非 `$bindable` 的 prop fallback 值不是响应式对象
6. **派生返回新引用** — 即便内容相同也会触发下游更新
7. **`$inspect` 是开发工具** — 生产编译为 noop，不可用于业务逻辑
8. **`$host` 仅 custom element** — 普通组件中使用会编译错误
9. **Map/Set 必须用响应式版本** — 原生 API 不会触发更新
10. **跨文件 export `$state`** — `export let` 重新赋值的 `$state` 不可行

## FAQ

**Q: `$derived` 和 `$state` 的区别？**
A: `$state` 创建可变状态；`$derived` 创建只读派生值，自动从依赖推导。5.25+ 可用 `let` 覆盖派生实现乐观 UI。

**Q: 什么时候用 `$effect`？**
A: 仅用于副作用：DOM 操作、第三方库调用、网络请求。派生值同步状态永远不用 `$effect`。

**Q: `$state.raw` 和普通 `$state` 的区别？**
A: `$state.raw` 不对数组/对象创建深层代理，性能更好，但只能通过重新赋值来更新。

**Q: `$props` 能解构吗？**
A: 可以，且支持默认值、重命名、rest 解构、$bindable、$props.id()。

**Q: `$bindable` 与函数绑定区别？**
A: `$bindable` 是声明 prop 可双向；函数绑定 `bind:value={() => v, setter}` 是自定义读写逻辑。两者可组合。

**Q: 何时用 `$bindable` vs callback props？**
A: 受控表单用 `$bindable`；事件通知用 callback props。避免滥用 `$bindable` 造成数据流混乱。

**Q: `$inspect` 在生产会运行吗？**
A: 不会，生产构建编译为 noop。

**Q: `$effect.pending()` 有什么用途？**
A: 显示当前 boundary 中待定的 Promise 数量，可用于全局 loading 状态。

**Q: `$effect.root()` 何时用？**
A: 测试工具、组件外创建 effect、模块级 effect。需手动调用返回的 destroy() 清理。

**Q: 跨模块共享状态用 `.svelte.js` 还是 Context？**
A: 真正全局单例用 `.svelte.js`；请求级隔离/组件树共享优先 Context。

## Examples

可执行的代码示例，见 `examples/` 目录：

| 文件 | 内容 |
|------|------|
| `state-patterns.md` | `$state` 基础、深层代理、`$state.raw`/`snapshot`/`eager`、类字段、内置响应式类、跨模块 |
| `derived-patterns.md` | `$derived` 基础、`$derived.by`、乐观 UI、解构派生、update propagation、async |
| `effect-patterns.md` | `$effect` 基础、cleanup、`$effect.pre`/`tracking`/`pending`/`root`、禁忌、untrack |
| `props-patterns.md` | `$props` 解构、类型安全、Rest Props、`$bindable`、`$props.id()`、泛型组件 |
| `bindable-patterns.md` | `$bindable` 全部用法：双向绑定、fallback、函数绑定、数组共享 |
| `inspect-patterns.md` | `$inspect`/`$inspect.trace`/`$inspect.with` 调试模式 |

## References

深入技术参考，见 `references/` 目录：

| 文件 | 内容 |
|------|------|
| `runes-overview.md` | Runes 整体架构、vs Legacy 对比、生命周期 |
| `$state-deep.md` | Proxy 行为、`$state.raw`/`snapshot`/`eager`、类中 `$state`、内置响应式类、跨模块 |
| `$derived-deep.md` | 表达式 vs `$derived.by`、依赖追踪、可写派生、Push-pull、解构 |
| `$effect-deep.md` | pre/tracking/cleanup/pending/root、追踪规则、常见错误 |
| `$props-deep.md` | 解构、Rest Props、Type safety、`$props.id()`、泛型 |
| `$bindable-deep.md` | 完整 `$bindable` 参考：双向绑定、fallback、函数绑定组合 |
| `$inspect-deep.md` | `$inspect`、`$inspect.with`、`$inspect.trace` 完整参考 |
| `$host-deep.md` | 自定义元素 `$host` 用法、`<svelte:options>` 配置、典型模式 |
| `context-deep.md` | createContext vs setContext、类型安全、SSR |
