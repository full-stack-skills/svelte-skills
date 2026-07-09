# $state 深层机制

## 概述

`$state(initial)` 是 Svelte 5 的核心响应式原语。它接收任意值，返回：
- 对数组/简单对象：**深度响应式 Proxy**（递归）
- 对原始值（number/string/boolean/...）：**Signal**（读写触发更新）
- 对类实例：**原样**返回（实例不代理）

`$state` 与 Svelte 4 的 `let` 隐式响应式相反：必须**显式**调用 `$state` 才会响应。

## Proxy 行为

```svelte
<script>
  let obj = $state({ user: { name: 'Alice', tags: ['a', 'b'] } });

  // ✅ 触发更新
  obj.user.name = 'Bob';
  obj.user.tags.push('c');
  obj.user.tags[0] = 'x';

  // ✅ 也触发（解构后单独更新）
  let { user } = obj;
  user.name = 'Charlie'; // 触发（通过 obj.user）
</script>
```

### 代理何时停止

Svelte 递归代理直到遇到：
- 类实例（不代理）
- `Object.create(null)` 或带自定义原型的对象
- 原始值（number/string/boolean/null/undefined）
- 不可代理的内部 slot（Map/Set/Date/URL —— 见下）

> 文档："State is proxified recursively until Svelte finds something other than an array or simple object (like a class or an object created with `Object.create`)."

### 修改属性**不**修改原对象

```js
let todos = $state([{ done: false, text: 'x' }]);
todos[0].done = true; // 内部创建新代理，原对象不动
```

如果想用自定义 Proxy 行为：先创建响应式代理，**再**用 `Proxy.revocable` 或第三方库包装。

## 内置响应式类（svelte/reactivity）

普通 `Map`/`Set`/`Date`/`URL` 不会响应。Svelte 提供响应式版本：

```ts
import {
  SvelteSet, SvelteMap, SvelteDate, SvelteURL,
  SvelteURLSearchParams
} from 'svelte/reactivity';

const tags = new SvelteSet<string>();
const cache = new SvelteMap<string, number>();
const lastSeen = new SvelteDate();
const url = new SvelteURL('https://example.com');
const params = new SvelteURLSearchParams();

tags.add('x');      // 触发更新
cache.set('a', 1);  // 触发更新
lastSeen.setTime(Date.now());
url.pathname = '/';
```

> 这些类实现了**与原生同名的 API** —— 可直接替换 `import { Map } from ...`。

## $state.raw（无代理）

适用于：大数据、高频更新但不需深层响应、外部 API 数据：

```svelte
<script>
  let data = $state.raw([]);

  // ❌ 不触发更新
  data.push({ id: 1 });

  // ✅ 触发更新
  data = [...data, { id: 1 }];
</script>
```

> "State declared with `$state.raw` cannot be mutated; it can only be reassigned."
> raw state **可以包含**响应式 state（如 `raw` 数组里放代理对象）。

## $state.snapshot（序列化）

用于将 Proxy 转为普通对象（发送给外部/JSON）：

```svelte
<script>
  let state = $state({ count: 0, items: ['a'] });

  function save() {
    // structuredClone 也接受 Proxy，但传外部库时建议 snapshot
    const plain = $state.snapshot(state);
    localStorage.setItem('state', JSON.stringify(plain));
  }
</script>
```

> 若值有 `toJSON()` 方法，snapshot 会克隆 `toJSON()` 的返回值而非原对象。

## $state.eager（即时值）

用于需要非异步获取当前值的场景（如 `aria-current`）：

```svelte
<script>
  let path = $state('/');

  // ✅ aria-current 在点击时立即计算
  let isActive = $state.eager(path) === '/';
</script>
```

> 用在 `await` 表达式前的"非 await 部分"。文档："In general, allowing Svelte to coordinate updates will provide a better user experience."

## $state 与类

```svelte
<script lang="ts">
  class Counter {
    count = $state(0);
    #private = $state(0);

    constructor(initial = 0) {
      this.count = initial; // 已用字段声明，构造器不能再 $state
    }

    increment() { this.count += 1; }
    reset = () => { this.count = 0; };  // 箭头函数字段 → this 自动绑定
  }

  let counter = new Counter();
</script>

<button onclick={counter.increment}>+</button>
<p>{counter.count}</p>
```

### `this` 绑定问题

```svelte
<!-- ❌ this 会是 button 元素 -->
<button onclick={counter.reset}>Reset</button>

<!-- ✅ 方案 1：内联函数 -->
<button onclick={() => counter.reset()}>Reset</button>

<!-- ✅ 方案 2：箭头函数字段 -->
class Counter {
  reset = () => { this.count = 0; };
}
```

### 编译器对 $state 字段的转换

> "The compiler transforms `done` and `text` into `get`/`set` methods on the class prototype referencing private fields. This means the properties are not enumerable."

转换大致等价于：

```js
class Todo {
  #done = $state(false);
  get done() { return this.#done; }
  set done(v) { this.#done = v; }
}
```

## 解构陷阱

```svelte
<script>
  let user = $state({ name: 'Ada', age: 30 });

  // ❌ 解构后再赋值不会影响 user
  let { name, age } = user;
  name = 'Bob'; // user.name 不变

  // ✅ 始终用代理属性
  function rename(n) { user.name = n; }
</script>
```

> "If you destructure a reactive value, the references are not reactive — as in normal JavaScript, they are evaluated at the point of destructuring."

## Passing state into functions

JavaScript 是**按值传递**：

```js
function add(a, b) { return a + b; }
let a = 1, b = 2;
let total = add(a, b); // 3
a = 3;
console.log(total); // 仍是 3
```

`$state` 同样按值传递 —— 函数拿到的是**当前值**快照。若需要"实时"读取，传入 getter：

```js
/**
 * @param {() => number} getA
 * @param {() => number} getB
 */
function add(getA, getB) {
  return () => getA() + getB();
}

let a = $state(1);
let b = $state(2);
let total = add(() => a, () => b);
console.log(total()); // 3
a = 3; b = 4;
console.log(total()); // 7
```

`getter` 也可来自 proxy 的属性 / `get` 访问器 / 函数引用 —— 任何能返回当前值的引用。

> 文档："Note that 'functions' is broad — it encompasses properties of proxies and get/set properties... though if you find yourself writing code like that, consider using classes instead."

## Passing state across modules

### 关键限制

`export let count = $state(0)` 会在另一文件读为**Signal 对象**而非数字（编译器无法跨文件包裹 `$.get`/`$.set`）。

```js
// state.svelte.js
export let count = $state(0); // ❌

// index.js
import { count } from './state.svelte.js';
console.log(typeof count); // 'object', not 'number'
```

### 解决方案 1：不可重新赋值的对象

```js
// counter.svelte.js
export const counter = $state({ count: 0 });
export function increment() { counter.count += 1; }
```

> Svelte 不会包 `counter.count += 1` 为 `$.set`，因为 `counter` 本身没重新赋值。

### 解决方案 2：模块内私有 + 函数导出

```js
let count = $state(0);
export function getCount() { return count; }
export function increment() { count += 1; }
```

### 跨请求 SSR 隔离

`.svelte.js` 模块级状态在 SSR 时每个请求创建新模块实例 —— 不会跨用户泄露。但同一用户多次访问可能复用（取决于打包器）。如需严格隔离用 Context API。

## 数组 / Set / Map 操作的响应性总结

| 操作 | `$state` 数组 | `$state.raw` 数组 | `SvelteSet` | `SvelteMap` |
|------|--------------|------------------|-------------|-------------|
| 索引赋值 | ✅ | ❌（需重新赋值） | — | — |
| push/pop/splice | ✅ | ❌ | ✅ | — |
| .add / .delete | — | — | ✅ | — |
| .set / .get | — | — | — | ✅ |
| 整体重新赋值 | ✅ | ✅ | ❌ | ❌ |

## 性能建议

1. **大数据集**用 `$state.raw` + 不可变更新
2. **频繁变化但不影响 UI** 的数据用普通 `let`
3. **类实例**用类字段 `$state` 而非包装成 `$state({ ... })`
4. **跨组件共享**优先 Context（请求隔离），仅全局单例用 `.svelte.js`
5. **避免** `$state` 巨型对象整体重新赋值 → 用局部变更
