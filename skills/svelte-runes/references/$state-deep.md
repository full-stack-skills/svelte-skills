# $state 深层机制

## Proxy 行为

`$state({...})` 和 `$state([...])` 返回深层 Proxy：

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

## $state.snapshot（序列化）

用于将 Proxy 转为普通对象（发送给外部/JSON）：

```svelte
<script>
  let state = $state({ count: 0, items: ['a'] });

  function save() {
    // fetch 不接受 Proxy
    const plain = $state.snapshot(state);
    localStorage.setItem('state', JSON.stringify(plain));
  }
</script>
```

## $state.eager（即时值）

用于需要非异步获取当前值的场景（如 `aria-current`）：

```svelte
<script>
  let path = $state('/');

  // ✅ aria-current 在点击时立即计算，不用等 async
  let isActive = $state.eager(path) === '/';
</script>
```

## $state 与类

```svelte
<script lang="ts">
  class Counter {
    count = $state(0);
    #private = $state(0);

    constructor(initial = 0) {
      this.count = initial;
    }

    increment() { this.count += 1; }
    reset = () => { this.count = 0; };
  }

  let counter = new Counter();
</script>

<button onclick={counter.increment}>+</button>
<p>{counter.count}</p>
<button onclick={counter.reset}>Reset</button>
```

## 跨组件共享状态边界

`.svelte.js` 模块级状态在 SSR 时的行为：
- 每个请求创建新模块实例
- 不会在用户间泄露
- 但同一用户多次访问可能复用（取决于打包工具）

如需严格隔离，用 Context API。
