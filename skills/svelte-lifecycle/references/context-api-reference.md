# Context API Reference

## 模块

```js
import {
  createContext, setContext, getContext, hasContext, getAllContexts
} from 'svelte';
```

## createContext（Svelte 5.40+）

**签名**：
```ts
function createContext<T>(): [get: () => T, set: (value: T) => T];
```

返回 `[get, set]` 一对函数。

```ts
// context.ts
import { createContext } from 'svelte';

interface User { name: string }
export const [getUser, setUser] = createContext<User>();
```

### get

```ts
function get(): T;
```

- 在 set 之后调用
- 若没有 set 过（不在 set 的子树中），Svelte 警告并返回 `undefined`

### set

```ts
function set(value: T): T;
```

- 必须在组件初始化期间调用
- 必须同步调用（不能在事件处理器或 $effect 中）
- 返回设置的值（方便链式使用）

## setContext / getContext（备选）

**签名**：
```ts
function setContext<T>(key: any, value: T): T;
function getContext<T>(key: any): T;
```

key 与 value 可为任意 JS 值。

```svelte
<!-- Parent -->
<script>
  import { setContext } from 'svelte';
  setContext('my-key', { foo: 1 });
</script>

<!-- Child -->
<script>
  import { getContext } from 'svelte';
  const v = getContext('my-key');
</script>
```

## hasContext

**签名**：`function hasContext(key: any): boolean`

```svelte
<script>
  import { hasContext, getContext } from 'svelte';
  const theme = hasContext('theme') ? getContext('theme') : 'light';
</script>
```

## getAllContexts

**签名**：`function getAllContexts(): Map<any, any>`

```svelte
<script>
  import { getAllContexts } from 'svelte';
  const all = getAllContexts();
  console.log([...all.entries()]);
</script>
```

返回当前组件上下文栈中所有 key → value。

## 调用时机约束

| 函数 | 何时调用 |
|---|---|
| `setContext` / `set` (from createContext) | 组件初始化期间，**同步** |
| `getContext` / `get` | 组件初始化或事件处理器（无时机限制） |
| `hasContext` | 任意时机 |
| `getAllContexts` | 任意时机 |

```svelte
<!-- ❌ 错误 -->
<script>
  import { setContext } from 'svelte';
  function handler() {
    setContext('k', 1);  // 错误！事件处理器内
  }
</script>
```

```svelte
<!-- ✅ 正确 -->
<script>
  import { setContext } from 'svelte';
  setContext('k', 1);  // 顶层
</script>
```

## 与 state 组合

### 对象（自动响应）

```svelte
<script>
  import { setContext } from 'svelte';
  const counter = $state({ count: 0 });
  setContext('counter', counter);
</script>
```

修改属性 → 响应式。

### 原始值（需传函数 getter）

```svelte
<script>
  import { setContext } from 'svelte';
  let count = $state(0);
  setContext('count', () => count);
</script>
```

```svelte
<!-- Child -->
<script>
  import { getContext } from 'svelte';
  const getCount = getContext('count');
</script>

<p>{getCount()}</p>
```

### 响应式对象 + 函数接口

```ts
// counter.ts
import { createContext } from 'svelte';
interface Counter { count: number; inc: () => void; }
export const [getCounter, setCounter] = createContext<Counter>();
```

```svelte
<!-- App -->
<script>
  import { setCounter } from './counter.ts';
  const counter = $state({ count: 0 });
  setCounter({
    get count() { return counter.count; },
    inc() { counter.count++; }
  });
</script>
```

## SSR 行为

每个请求渲染独立组件树，**context 不会跨请求共享**。这是 Context 替代模块级 `$state` 的核心原因：

```svelte
<!-- ❌ 模块级 $state 在 SSR 下会在请求间共享 -->
<!-- user.svelte.ts -->
export const user = $state({ name: '' });
```

```svelte
<!-- ✅ Context 隔离每个请求 -->
<script>
  import { setContext } from 'svelte';
  let { data } = $props();
  setContext('user', $state({ name: data.user.name }));
</script>
```

## Component Testing

Svelte 5.49+ 起，可以用普通函数包装：

```js
import { mount, unmount } from 'svelte';
import { expect, test } from 'vitest';
import { setUser } from './context';
import MyComponent from './MyComponent.svelte';

test('MyComponent', () => {
  function Wrapper(...args) {
    setUser({ name: 'Bob' });
    return MyComponent(...args);
  }
  const c = mount(Wrapper, { target: document.body });
  expect(document.body.innerHTML).toBe('<h1>Hello Bob!</h1>');
  unmount(c);
});
```

> 该方法也适用于 `hydrate` 和 `render`。

## 最佳实践

- 用 `createContext`（避免 key 冲突，享受类型）
- 不要重新赋值 set 过的 `$state` 对象
- 原始值用函数 getter
- 涉及用户特定数据用 Context，不用模块级 state
- 库作者：暴露 `set`/`get` 对给消费者使用
