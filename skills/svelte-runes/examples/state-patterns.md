# $state Patterns

## 1. 基础用法

```svelte
<script>
  let count = $state(0);
  let name = $state('world');
  let user = $state({ name: 'Alice', age: 30 });
</script>

<button onclick={() => count++}>+1</button>
<p>count: {count}</p>
```

## 2. 深层代理

```svelte
<script>
  let todos = $state([
    { done: false, text: 'buy milk' },
    { done: false, text: 'walk dog' }
  ]);

  function toggle(i) {
    todos[i].done = !todos[i].done;
  }

  function add(text) {
    todos.push({ done: false, text });
  }
</script>
```

> 编译器会递归代理数组与普通对象，遇到类实例或 `Object.create` 出来的对象则停止。

## 3. $state.raw（避免代理开销）

```svelte
<script>
  // 大数据量、不需要深层代理时
  let bigData = $state.raw([]);

  // 只能重新赋值触发更新
  function loadData(newData) {
    bigData = newData; // ✅
    bigData.push(...); // ❌ 不触发
  }
</script>
```

`$state.raw` 内部仍可包含响应式状态（如原始数组里放代理对象）。

## 4. $state.snapshot（传外部 API）

```svelte
<script>
  let proxy = $state({ count: 0, items: ['a', 'b'] });

  function sendToServer() {
    // 外部 API 不接受 Proxy
    const snapshot = $state.snapshot(proxy);
    fetch('/api', {
      method: 'POST',
      body: JSON.stringify(snapshot)
    });
  }
</script>
```

若值有 `toJSON` 方法，snapshot 会克隆 `toJSON` 的返回值。

## 5. $state.eager（即时 UI 更新）

```svelte
<script>
  let path = $state('/');

  // 用于异步边界外（如导航状态）
  // 让点击立即反馈，不用等 async
</script>

<nav>
  <a href="/" aria-current={$state.eager(path) === '/' ? 'page' : null}>Home</a>
  <a href="/about" aria-current={$state.eager(path) === '/about' ? 'page' : null}>About</a>
</nav>
```

谨慎使用 —— 仅用于用户操作后的即时反馈。

## 6. 类字段中的 $state

```svelte
<!-- Counter.svelte -->
<script lang="ts">
  class Counter {
    count = $state(0);
    #value = $state(0); // 私有

    constructor(initial = 0) {
      this.value = $state(initial);
    }

    increment() {
      this.count += 1;
    }

    reset = () => {
      this.count = 0;
      this.#value = 0;
    };
  }

  let counter = new Counter();
</script>

<button onclick={counter.increment}>+</button>
<p>Count: {counter.count}</p>
<button onclick={counter.reset}>Reset</button>
```

编译器将 `$state` 字段转成原型上的 `get`/`set`，因此这些属性**不可枚举**。注意 `this` 绑定问题：方法直接传给事件处理器会丢失 `this`，请用箭头函数字段或内联 `() => todo.reset()`。

## 7. 跨模块 $state（.svelte.js）

```js
// state/counter.svelte.js
export const counterState = $state({
  count: 0,
  increment() {
    this.count += 1;
  }
});

// ComponentA.svelte
<script>
  import { counterState } from '$lib/state/counter.svelte.js';
</script>
<button onclick={counterState.increment}>+</button>

// ComponentB.svelte
<script>
  import { counterState } from '$lib/state/counter.svelte.js';
</script>
<p>Count: {counterState.count}</p>
```

关键约束见下两条。

## 8. 嵌套状态对象

```svelte
<script>
  let store = $state({
    user: {
      profile: {
        name: 'Alice',
        settings: { theme: 'dark' }
      }
    },
    posts: []
  });

  // 深层变更自动追踪
  function updateTheme(theme) {
    store.user.profile.settings.theme = theme; // ✅
  }

  function addPost(post) {
    store.posts.push(post); // ✅
  }
</script>
```

## 9. 类中字段 + 构造器（编译器约定）

`$state` 字段可以是 public 或 private（`#` 前缀）。构造器中**第一次**对属性赋值也可使用 `$state`：

```js
class Todo {
  done = $state(false);

  constructor(text) {
    this.text = $state(text); // 第一次赋值 OK
  }

  // 箭头函数字段 → 自动绑定 this
  toggle = () => {
    this.done = !this.done;
  };
}

const todo = new Todo('learn runes');
// <button onclick={todo.toggle}>toggle</button>  // ✅
```

> 编译器把 `done`/`text` 转换为原型上的 `get`/`set` 并指向私有字段。

## 10. $state + 内置响应式类（svelte/reactivity）

Svelte 提供 `Set`/`Map`/`Date`/`URL` 的响应式版本，导入自 `svelte/reactivity`：

```svelte
<script>
  import { SvelteSet, SvelteMap, SvelteDate, SvelteURL } from 'svelte/reactivity';

  const tags = new SvelteSet(['a', 'b']);
  const cache = new SvelteMap();
  const lastSeen = new SvelteDate();
  const url = new SvelteURL('https://example.com');

  function addTag(t) {
    tags.add(t);     // ✅ 触发更新
    cache.set(t, 1); // ✅ 触发更新
    lastSeen.setTime(Date.now());
    url.pathname = '/new';
  }
</script>

<p>tags: {tags.size}</p>
<p>last seen: {lastSeen.toLocaleString()}</p>
<p>url: {url.href}</p>
```

## 11. $state + 普通 Map/Set 的陷阱

直接用原生 `Map`/`Set` 包裹在 `$state` 里**不会**被代理 —— `.set()` 不会触发更新：

```svelte
<script>
  // ❌ 不会触发更新
  const broken = $state(new Map());
  broken.set('a', 1); // 静默失败

  // ✅ 用 SvelteMap
  import { SvelteMap } from 'svelte/reactivity';
  const ok = new SvelteMap();
  ok.set('a', 1); // 触发更新
</script>
```

## 12. 将 state 传入函数 —— getter 函数模式

`$state` 是按值传递：函数拿到的是**当前值**的快照。若函数需要读取最新值，传入 getter：

```js
// utils/math.js
/** @param {() => number} getA @param {() => number} getB */
export function add(getA, getB) {
  return () => getA() + getB();
}
```

```svelte
<script>
  import { add } from './utils/math.js';

  let a = $state(1);
  let b = $state(2);
  const total = add(() => a, () => b);

  $: console.log(total()); // 3，然后随 a/b 变化
</script>
```

更地道的做法：直接传 `getter` 或包装为 `get` 属性，避免到处传函数。

## 13. 跨模块导出 $state 的两种安全模式

**模式 A：导出不可重新赋值的对象**（推荐）

```js
// counter.svelte.js
export const counter = $state({ count: 0 });
export function increment() { counter.count += 1; }
```

**模式 B：模块内私有 + 导出函数**

```js
// counter.svelte.js
let count = $state(0);
export function getCount() { return count; }
export function increment() { count += 1; }
```

**错误写法**（编译器无法跨文件包裹 getter/setter）：

```js
// ❌ 另一文件读到的会是 Signal 对象，不是数值
export let count = $state(0);
```

## 14. 解构陷阱与替代方案

```svelte
<script>
  let user = $state({ name: 'Ada', age: 30 });

  // ❌ 重新赋值解构变量不会影响 user
  let { name, age } = user;
  name = 'Bob'; // user.name 不变

  // ✅ 始终访问代理属性
  function rename(n) { user.name = n; }
</script>
```

如果需要派生"取局部字段"，用 `$derived`：

```svelte
<script>
  let user = $state({ name: 'Ada', age: 30 });
  let name = $derived(user.name);
  let age = $derived(user.age);
</script>
```

## 15. 用 `toJSON` 自定义 snapshot 形状

```svelte
<script>
  class Point {
    x = $state(0);
    y = $state(0);
    toJSON() {
      return { coordinates: [this.x, this.y] };
    }
  }

  const p = $state(new Point());
  p.x = 3; p.y = 4;
  $inspect($state.snapshot(p));
  // → { coordinates: [3, 4] }（不是 { x: 3, y: 4 }）
</script>
```

## 16. 数组/对象的响应式操作清单

```svelte
<script>
  let list = $state([1, 2, 3]);
  let dict = $state({ a: 1 });

  // ✅ 触发
  list[0] = 99;
  list.push(4);
  list.splice(1, 1, 20);
  list.sort();
  list.length = 0;          // 清空（写入 length）

  dict.a = 2;
  delete dict.a;            // ⚠️ Svelte 5 Proxy 上 delete 会触发更新
  Object.assign(dict, { b: 3 });
</script>
```

`$state.raw` 同样只支持**重新赋值**整个对象/数组来触发更新。
