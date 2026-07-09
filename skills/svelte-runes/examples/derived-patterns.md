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

## 8. 派生与异步

```svelte
<script>
  let query = $state('');

  // $derived 中的 async 需注意：
  // await 之后的同步读取也是依赖
  let results = $derived.by(async () => {
    if (!query) return [];
    const res = await fetch(`/api?q=${query}`);
    return res.json();
  });
</script>
```
