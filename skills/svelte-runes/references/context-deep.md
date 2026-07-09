# Context API 深层参考

## createContext（类型安全，推荐）

```ts
// context.ts
import { createContext } from 'svelte';

export const [getUser, setUser] = createContext<{
  name: string;
  age: number;
}>();
```

```svelte
<!-- Parent.svelte -->
<script>
  import { setUser } from './context';
  setUser({ name: 'Alice', age: 30 });
</script>
```

```svelte
<!-- Child.svelte -->
<script>
  import { getUser } from './context';
  const user = getUser();
</script>
```

## setContext / getContext（无类型）

```svelte
<!-- Parent.svelte -->
<script>
  import { setContext } from 'svelte';
  setContext('theme', 'dark');
  setContext('user', { name: 'Bob' });
</script>
```

```svelte
<!-- Child.svelte -->
<script>
  import { getContext } from 'svelte';
  const theme = getContext('theme');
  const user = getContext('user');
</script>
```

## 响应式 Context

Context 本身不是响应式的——它存储的值可以是响应式的：

```svelte
<!-- Parent.svelte -->
<script>
  import { setCounter } from './context';
  let count = $state(0);
  setCounter({ count });
</script>
```

```svelte
<!-- Child.svelte -->
<script>
  import { getCounter } from './context';
  const counter = getCounter();
</script>

<p>{counter.count}</p>
```

⚠️ **重要**：不要重新赋值整个对象：
```svelte
<!-- ❌ 错误：断开链接 -->
<button onclick={() => counter = { count: 0 }}>Reset</button>

<!-- ✅ 正确：修改属性 -->
<button onclick={() => counter.count = 0}>Reset</button>
```

## SSR 注意事项

Context 在 SSR 时按请求隔离，不会跨用户泄露。

## 替代方案对比

| 方案 | 适用场景 |
|------|---------|
| `.svelte.js` 模块 | 真正全局的、跨请求共享的状态 |
| Context API | 组件树内共享、请求级隔离 |
| Props drilling | 简单、浅层传递 |

优先用 Context，比模块级状态更安全。
