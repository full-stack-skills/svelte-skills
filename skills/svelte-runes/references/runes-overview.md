# Runes 整体架构

## Runes vs Legacy（Svelte 4）对比

| 特性 | Legacy 模式 | Runes 模式 |
|------|------------|-----------|
| 响应式变量 | `let count = 0`（隐式） | `let count = $state(0)` |
| 派生值 | `$: doubled = count * 2` | `let doubled = $derived(count * 2)` |
| 副作用 | `$: { console.log(count); }` | `$effect(() => { console.log(count); })` |
| Props | `export let name` | `let { name } = $props()` |
| 可变 Props | 不支持 | `let { value = $bindable() } = $props()` |
| 状态来源 | `let` 声明位置 | `$state()` 调用位置 |
| 模块级状态 | store | `.svelte.js` 文件 + `$state` |

## Runes 生命周期

```
$state → 初始化
    ↓
$derived → 自动计算（依赖变化时重算）
    ↓
$effect → 副作用（依赖变化时重跑）
    ↓
cleanup（返回函数）→ 下次 effect 重跑前执行
```

## 模块级 $state（.svelte.js）

```js
// state/counter.svelte.js
export const counter = $state({ count: 0 });

// Counter.svelte
<script>
  import { counter } from '$lib/state/counter.svelte.js';
</script>

<button onclick={() => counter.count++}>+</button>
```

关键约束：
- `.svelte.js` 文件中的 `$state` 是模块级单例
- SSR 时每个请求有独立实例（防止状态泄露）
- 仅在真正需要跨组件共享时才用，优先用 Context

## Runes 不在的地方

以下位置不能用 Runes（会编译错误）：
- 普通 `.js`/`.ts` 文件（除非文件名以 `.svelte.js` 结尾）
- `<script module>` 中（可以用 `$derived` 在模板引用前定义）
- 普通 HTML/JS 文件

## $state.proxy 行为

```svelte
<script>
  let obj = $state({ a: { b: 1 } });

  // ✅ 深层响应
  obj.a.b = 2;

  // ✅ 数组方法响应
  obj.items.push(4);

  // ⚠️ 重新赋值才触发
  obj = { ...obj, a: { b: 3 } };
</script>
```

## $effect 依赖追踪规则

只有**同步读取**的 `$state` 才算依赖：

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  $effect(() => {
    console.log(a); // 依赖 a

    setTimeout(() => {
      console.log(b); // 不算依赖（异步）
    }, 100);
  });
</script>
```
