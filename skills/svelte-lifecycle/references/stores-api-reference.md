# Stores API Reference

## 模块

```js
import {
  writable, readable, derived, readonly, get
} from 'svelte/store';
```

## Store 契约

任何对象只要实现 `subscribe` 方法就是 store：

```ts
type Subscriber<T> = (value: T) => void;
type Unsubscriber = () => void;
type Updater<T> = (value: T) => T;

interface Readable<T> {
  subscribe(run: Subscriber<T>, invalidate?: (value?: T) => void): Unsubscriber;
}

interface Writable<T> extends Readable<T> {
  set(value: T): void;
  update(updater: Updater<T>): void;
}
```

### 契约规则

1. `subscribe(fn)` 必须**同步**用当前值调用 `fn`。
2. 每次值变化，**同步**调用所有 active 订阅。
3. `subscribe` 返回 unsubscribe 函数。
4. 可选 `set` 方法，写入新值。
5. 为兼容 RxJS，`subscribe` 也可返回 `{ unsubscribe() {} }`。

## writable

**签名**：
```ts
function writable<T>(value: T, start?: (set: Updater<T>, update: Updater<T>) => Unsubscriber): Writable<T>;
```

```js
import { writable } from 'svelte/store';

const store = writable(0);
store.set(1);
store.update(n => n + 1);

// start/stop
const s = writable(0, (set, update) => {
  console.log('first subscriber');
  return () => console.log('no subscribers');
});
```

- `set(value)`：值不变则不通知
- `update(fn)`：fn 接收旧值，返回新值

## readable

**签名**：
```ts
function readable<T>(value: T, start?: (set: (v: T) => void, update: (fn: Updater<T>) => void) => Unsubscriber): Readable<T>;
```

```js
import { readable } from 'svelte/store';

const time = readable(new Date(), (set) => {
  set(new Date());
  const id = setInterval(() => set(new Date()), 1000);
  return () => clearInterval(id);
});
```

外部不可调用 `set`/`update`（方法未导出）。

## derived

**签名**：
```ts
function derived<S, T>(
  stores: S | S[],
  fn: (values: S, set: (v: T) => void, update: (fn: (v: T) => T) => void) => T | Unsubscriber | void,
  initial?: T
): Readable<T>;
```

- 回调首次在第一个订阅者订阅时执行
- 依赖变化时再次执行
- 回调可返回 cleanup 函数，下次执行前或最后订阅者退订时调用
- 回调签名：`(values, set, update) => T | (() => void)`
- 若不立即返回值，可使用 `set` / `update` 在异步中赋值
- 第三个参数为初始值；不传则在第一次 `set`/`update` 前为 `undefined`

```js
import { derived, writable } from 'svelte/store';

const a = writable(1);
const b = writable(2);

// 同步
const sum = derived([a, b], ([$a, $b]) => $a + $b);

// 异步 + 初始值
const delayed = derived(
  a,
  ($a, set) => {
    setTimeout(() => set($a * 2), 1000);
  },
  0
);

// 数组形式
const sum2 = derived([a, b], ([$a, $b]) => $a + $b);
```

## readonly

**签名**：`readonly<T>(store: Readable<T>): Readable<T>`

```js
import { readonly, writable } from 'svelte/store';

const w = writable(1);
const r = readonly(w);
// r.set  // undefined —— TypeScript 报错
r.subscribe(console.log);
w.set(2);
```

## get

**签名**：`get<T>(store: Readable<T>): T`

```js
import { get, writable } from 'svelte/store';
const s = writable(5);
console.log(get(s));  // 5
```

> 通过临时订阅读取值，**热路径开销大**——不要在循环或 render 中调用。

## $store 语法糖

```svelte
<script>
  import { writable } from 'svelte/store';
  const count = writable(0);
</script>

<p>{$count}</p>            <!-- 自动订阅 -->
<button onclick={() => $count++}>inc</button>  <!-- 调用 .set -->
```

约束：
- store 必须在**组件顶层**声明
- 非 store 变量不要加 `$` 前缀
- 赋值给 `$store` 会调用 `.set`

## 自定义 store

```ts
function createCounter() {
  let count = 0;
  const subscribers = new Set<(n: number) => void>();

  return {
    subscribe(fn: (n: number) => void) {
      subscribers.add(fn);
      fn(count);
      return () => subscribers.delete(fn);
    },
    set(n: number) {
      count = n;
      subscribers.forEach(fn => fn(n));
    },
    update(fn: (n: number) => number) {
      this.set(fn(count));
    }
  };
}
```

满足 store 契约即可与 `$store` 语法糖、`derived`、`readonly` 等互操作。
