# $derived Patterns

## 1. 基础用法

```svelte
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
  let isBig = $derived(count > 10);
</script>

<p>doubled: {doubled} (自动更新)</p>
<p>isBig: {isBig}</p>
```

> 派生表达式**必须**纯净：内部不能修改 `$state`（编译器会报错）。

## 2. $derived.by（复杂逻辑）

```svelte
<script>
  let items = $state([
    { name: 'Apple', price: 3.5, qty: 2 },
    { name: 'Banana', price: 1.2, qty: 5 },
    { name: 'Cherry', price: 8.0, qty: 1 }
  ]);

  let summary = $derived.by(() => {
    const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);
    const count = items.reduce((n, i) => n + i.qty, 0);
    return { total: total.toFixed(2), count };
  });

  let expensive = $derived.by(() =>
    items.filter(i => i.price > 5)
  );
</script>

<p>Total: ${summary.total} ({summary.count} items)</p>
<p>Expensive items: {expensive.map(i => i.name).join(', ')}</p>
```

`$derived(expr)` 实质等价于 `$derived.by(() => expr)`。

## 3. 解构派生

```svelte
<script>
  let user = $state({ first: 'John', last: 'Doe', age: 30 });

  // 解构后每个都是独立的 $derived
  let { first, last } = $derived(user);

  // 等价于：
  // let first = $derived(user.first);
  // let last = $derived(user.last);
</script>

<p>{first} {last}</p>
```

## 4. 派生值覆盖（乐观 UI，Svelte 5.25+）

```svelte
<script>
  let { post, like } = $props();
  let likes = $derived(post.likes);

  async function handleLike() {
    // 即时乐观更新
    likes += 1;
    try {
      await like();
    } catch {
      likes -= 1; // 回滚
    }
  }
</script>

<button onclick={handleLike}>❤️ {likes}</button>
```

声明为 `let` 才能重新赋值；`const` 派生只读。

## 5. 派生与数组方法

```svelte
<script>
  let numbers = $state([3, 1, 4, 1, 5, 9, 2, 6]);

  let sorted = $derived([...numbers].sort((a, b) => a - b));
  let sum = $derived(numbers.reduce((a, b) => a + b, 0));
  let avg = $derived(sum / numbers.length);
  let hasEven = $derived(numbers.some(n => n % 2 === 0));
</script>
```

## 6. 派生作为 props 依赖

```svelte
<script lang="ts">
  let {
    items,
    filter = $bindable('all')
  }: {
    items: Array<{ id: number; name: string; done: boolean }>;
    filter?: string;
  } = $props();

  let filtered = $derived(
    filter === 'all' ? items :
    filter === 'done' ? items.filter(i => i.done) :
    items.filter(i => !i.done)
  );
</script>
```

## 7. 派生中的条件逻辑

```svelte
<script>
  let age = $state(25);
  let isAdult = $derived(age >= 18);
  let category = $derived.by(() => {
    if (age < 13) return 'child';
    if (age < 20) return 'teenager';
    return 'adult';
  });
</script>
```

> 条件分支：派生**只追踪上次运行时实际读取的状态**。例如 `if (show) { return count }` 不会追踪 `count` 当 `show` 为 `false` 时。

## 8. 派生与异步（await 后也追踪）

```svelte
<script>
  let query = $state('');

  // await 之后的同步读取仍是依赖
  let results = $derived.by(async () => {
    if (!query) return [];
    const res = await fetch(`/api?q=${query}`);
    return res.json();
  });
</script>

{#await results}
  <p>loading…</p>
{:then items}
  <ul>{#each items as i}<li>{i.name}</li>{/each}</ul>
{:catch e}
  <p>error: {e.message}</p>
{/await}
```

> 仅 `$derived`/`$effect` 表达式自身的 `await` 会追踪；调用函数中的 `await` 不计入。

## 9. update propagation（push-pull）

派生用"推-拉"模型：依赖变化时**标记为脏**，真正被读取时**才重算**。若新值与旧值引用相同，下游不会更新：

```svelte
<script>
  let count = $state(0);
  let large = $derived(count > 10); // 布尔值
</script>

<button onclick={() => count++}>
  <!-- 按钮文本只在 large 变化时重渲染 -->
  {large}
</button>
```

> 派生返回值若是**新对象**，即便内容相同也会触发下游 —— 因此派生里要避免 `() => ({})` 这类写法。

## 10. 用 untrack 排除依赖

```svelte
<script>
  import { untrack } from 'svelte';

  let count = $state(0);
  let theme = $state('light');

  // 只追踪 count
  let displayCount = $derived.by(() => {
    const t = untrack(() => theme);
    return `${t}: ${count}`;
  });
</script>
```

`untrack` 在派生和 effect 中同样适用。

## 11. 派生与代理：mutate selected 改 items

`$derived` 不会把返回值包成代理 —— 它**保留原引用**。所以派生出来再修改属性，会影响到底层的 `$state`：

```svelte
<script>
  let items = $state([{ id: 1, name: 'foo' }]);
  let index = $state(0);
  let selected = $derived(items[index]);

  function rename() {
    selected.name = 'bar'; // ✅ 实际改的是 items[0].name
  }
</script>
```

## 12. 用 get/set 派生模拟可写计算属性

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  let sum = $derived({
    get value() { return a + b; },
    set value(v) { a = v - b; }
  });
</script>

<button onclick={() => sum.value = 10}>set a=8</button>
<p>sum: {sum.value}</p>
```

## 13. 派生链与调试

```svelte
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
  let quadrupled = $derived(doubled * 2);
  let isPositive = $derived(quadrupled > 0);

  // 任一上游变化都会沿链 dirty-mark，按需重算
</script>
```

调试：`$inspect` 或 `$inspect.trace()` 配合 effect/derived 使用。

## 14. 派生中不要做的事

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  // ❌ 表达式内有副作用
  // let sum = $derived((a = 0, b));  // 编译器报错

  // ❌ 派生里读 $effect
  // let result = $derived.by(() => {
  //   let captured;
  //   $effect(() => { captured = a; });  // 不要这样做
  //   return captured;
  // });
</script>
```

## 15. 何时用 $derived vs $state

| 场景 | 选 |
|------|----|
| 表达"X = f(Y)" | `$derived` |
| 表达"按下按钮后 Y = X + 1" | `$state`（手动更新）或 `$derived` + 重新赋值（5.25+） |
| 需要乐观 UI 回滚 | `$derived`（用 `let`） |
| 跨多个组件共享 | `$state`（封装在类或 `.svelte.js`） |

## 16. 解构 + 默认值

```svelte
<script>
  function getConfig() {
    return { host: '', port: 0, ssl: false };
  }
  let config = $state(getConfig());

  // 解构并设默认值
  let { host, port = 443, ssl = true } = $derived(config);
</script>
```

`$derived` 的解构与对象字面量解构语法完全一致。
