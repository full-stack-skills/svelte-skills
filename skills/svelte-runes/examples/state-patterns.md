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
