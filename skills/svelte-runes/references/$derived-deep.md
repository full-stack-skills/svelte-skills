# $derived 深层机制

## 表达式 vs $derived.by

```svelte
<script>
  let a = $state(3);
  let b = $state(4);

  // ✅ 简单表达式
  let sum = $derived(a + b);

  // ✅ 复杂逻辑用 $derived.by
  let factorial = $derived.by(() => {
    if (a <= 1) return 1;
    let result = 1;
    for (let i = 2; i <= a; i++) result *= i;
    return result;
  });
</script>
```

## 自动依赖追踪

```svelte
<script>
  let a = $state(1);
  let b = $state(2);
  let show = $state(true);

  // 只追踪 a（b 没在表达式中）
  let halfOfA = $derived(a / 2);

  // 追踪 a 和 b（条件中读到了 a）
  let result = $derived(show ? a + b : 0);
</script>
```

## 解构派生

```svelte
<script>
  let user = $state({ first: 'John', last: 'Doe', age: 30 });

  // 解构后每个字段独立派生
  let { first, last } = $derived(user);

  // 等价于：
  // let first = $derived(user.first);
  // let last = $derived(user.last);
</script>

<p>{first} {last}</p>
```

## 派生值可写

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  // 可写的派生值
  let sum = $derived({
    get value() { return a + b; },
    set value(v) { a = v - b; }
  });

  sum.value = 10; // a 变为 8
</script>
```

## 派生中不要做的事

❌ **不要用 $effect 同步派生值**：
```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  // ❌ 错误
  let sum;
  $effect(() => { sum = a + b; });

  // ✅ 正确
  let sum = $derived(a + b);
</script>
```

## $derived 中使用 $state（罕见）

```svelte
<script>
  let items = $state([1, 2, 3]);

  // 需要内部可变状态时
  let processed = $derived.by(() => {
    let cache = $state({ seen: new Set() });
    return items.map(x => {
      cache.seen.add(x);
      return x * 2;
    });
  });
</script>
```
