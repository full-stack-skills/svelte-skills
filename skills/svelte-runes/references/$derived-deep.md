# $derived 深层机制

## 概述

`$derived(expression)` 与 `$derived.by(() => ...)` 声明**派生状态**：基于其他响应式值的只读（默认）计算值。依赖变化时自动标脏，下次读取时按需重算。

```svelte
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
</script>
```

> 表达式必须**纯净** —— Svelte 禁止内部修改 `$state`（会编译报错）。

## 表达式 vs $derived.by

```svelte
<script>
  let a = $state(3);
  let b = $state(4);

  // ✅ 简单表达式
  let sum = $derived(a + b);

  // ✅ 复杂逻辑 → $derived.by
  let factorial = $derived.by(() => {
    if (a <= 1) return 1;
    let result = 1;
    for (let i = 2; i <= a; i++) result *= i;
    return result;
  });
</script>
```

> `$derived(expr)` 实质等价于 `$derived.by(() => expr)`。

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

### 依赖追踪规则

- 仅追踪**同步**读取的状态
- 调用**函数**内部读取的状态也会被追踪（除非用 `untrack`）
- `await` 之后**自身表达式**中的同步读取也是依赖
- 调用函数中的 `await` 内的读取**不**追踪

```js
let a = Promise.resolve(1);
let b = 2;
// a 和 b 都是依赖
let total = $derived(await a + b);
```

> 文档："This does not apply to await in functions that are called by the expression, only the expression itself."

## Overriding derived values（Svelte 5.25+）

派生默认只读，但 `let` 声明可**临时覆盖**（用于乐观 UI）：

```svelte
<script>
  let { post, like } = $props();

  let likes = $derived(post.likes);

  async function onclick() {
    // 立即乐观更新
    likes += 1;
    try {
      await like();
    } catch {
      // 失败回滚
      likes -= 1;
    }
  }
</script>

<button {onclick}>🧡 {likes}</button>
```

`const` 派生的覆盖会编译报错。

> 文档："Derived expressions are recalculated when their dependencies change, but you can temporarily override their values by reassigning them (unless they are declared with const)."

## Update propagation（推-拉模型）

Svelte 用 **push-pull reactivity**：
- **Push** —— 状态变化时立即通知所有依赖（无论直接/间接）
- **Pull** —— 派生值**只在被读取时**才重新计算

### 引用相同则下游不更新

```svelte
<script>
  let count = $state(0);
  let large = $derived(count > 10); // 布尔
</script>

<button onclick={() => count++}>
  <!-- 仅当 large 变化时按钮文本重渲染 -->
  {large}
</button>
```

> 派生返回**新对象**时，即便内容相同也会触发下游（不同引用）。派生内避免 `[1,2,3].map(...)` 这类每次返回新数组的写法 —— 会让下游始终重算。

## Deriveds and reactivity（不包代理）

与 `$state` 不同，`$derived` 不会把返回值包成 Proxy：

```js
// @errors: 7005
let items = $state([ /*...*/ ]);

let index = $state(0);
let selected = $derived(items[index]);

selected.name = 'X'; // ✅ 改底层 items[index].name
```

> "If items was not deeply reactive, mutating selected would have no effect."

## Destructuring

解构派生声明 → 每个变量独立派生：

```js
function stuff() { return { a: 1, b: 2, c: 3 } }
// ---cut---
let { a, b, c } = $derived(stuff());
// 等价于：
let _stuff = $derived(stuff());
let a = $derived(_stuff.a);
let b = $derived(_stuff.b);
let c = $derived(_stuff.c);
```

> 文档："the resulting variables will all be reactive."

## 派生中谨慎使用 $state

```svelte
<script>
  let items = $state([1, 2, 3]);

  // 罕见：需要内部缓存
  let processed = $derived.by(() => {
    let cache = $state({ seen: new Set() });
    return items.map(x => {
      cache.seen.add(x);
      return x * 2;
    });
  });
</script>
```

## 派生中不要做的事

### ❌ 不要用 $effect 同步派生值

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  // ❌
  let sum;
  $effect(() => { sum = a + b; });

  // ✅
  let sum = $derived(a + b);
</script>
```

### ❌ 派生内做副作用

派生是**纯净**计算 —— 任何 `console.log`、`fetch`、修改其他 `$state` 都会编译报错或产生未定义行为。

### ❌ 在派生中调用 $effect

```js
// ❌ 错误
let result = $derived.by(() => {
  let captured;
  $effect(() => { captured = a; });
  return captured;
});
```

## 用 untrack 排除依赖

```svelte
<script>
  import { untrack } from 'svelte';

  let count = $state(0);
  let theme = $state('light');

  // 只追踪 count
  let display = $derived.by(() => {
    const t = untrack(() => theme);
    return `${t}: ${count}`;
  });
</script>
```

> 文档："To exempt a piece of state from being treated as a dependency, use untrack."

## 用 get/set 派生模拟可写计算属性

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  let sum = $derived({
    get value() { return a + b; },
    set value(v) { a = v - b; }
  });

  // sum.value = 10 → a = 8
</script>
```

## 派生链与缓存

```svelte
<script>
  let a = $state(0);
  let b = $derived(a + 1);
  let c = $derived(b * 2);
  // a → b → c，任一上游变化会沿链 dirty-mark
</script>
```

派生按需缓存：未被读取时不会重算。读取顺序影响：若先读 `b` 再读 `c`，`b` 只重算一次。

## 派生与 $inspect.trace

```svelte
<script>
  let count = $state(0);

  let doubled = $derived.by(() => {
    $inspect.trace();
    return count * 2;
  });
</script>
```

`$inspect.trace` 必须是函数体的第一条语句（Svelte 5.14+）。当 derived 被标脏/重算时打印触发的状态。
