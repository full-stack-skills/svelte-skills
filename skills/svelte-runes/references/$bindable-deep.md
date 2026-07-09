# $bindable 完整参考

## 概述

`$bindable()` 是 `$props()` 的一个值标记 —— 表示该 prop 可以**双向绑定**。

```svelte
<!-- FancyInput.svelte -->
<script>
  let { value = $bindable(), ...props } = $props();
</script>

<input bind:value={value} {...props} />
```

> 文档："In Svelte, component props can be bound, which means that data can also flow up from child to parent. This isn't something you should do often — overuse can make your data flow unpredictable and your components harder to maintain — but it can simplify your code if used sparingly and carefully."

## 语法

```svelte
let {
  value = $bindable(),         // 必填
  other = $bindable('hello'),  // 带 fallback
  readonly = $bindable()       // bindable 但无 fallback
} = $props();
```

- `$bindable()` / `$bindable(value)` 只能用在解构的**默认值**位置
- 类型系统：`bindable<T>(value?: T): T`

## 父组件两种用法

```svelte
<!-- App.svelte -->
<script>
  import FancyInput from './FancyInput.svelte';
  let message = $state('hello');
</script>

<!-- 双向：父监听子 -->
<FancyInput bind:value={message} />

<!-- 单向：父不监听 -->
<FancyInput value={message} />
```

> 文档："The parent component doesn't have to use bind: — it can just pass a normal prop. Some parents don't want to listen to what their children have to say."

## Fallback 值

```js
let { value = $bindable('fallback') } = $props();
```

当父组件**完全不传** `value` 时使用 fallback。

## 何时使用 $bindable

| 场景 | 推荐 |
|------|------|
| 受控表单组件（input/checkbox/select） | `$bindable` |
| 子组件需要"暂时"覆盖 prop（unsaved state） | 临时重新赋值普通 prop |
| 子→父状态通知 | callback props |
| 跨组件状态共享 | Context / store |

## 与 mutation 的关系

`$bindable` 允许**两种**子→父通信方式：
1. **重新赋值**（`value = newValue`）
2. **变异共享代理**（`item.name = 'x'` —— 仅当父传的是 `$state` 代理时）

> 文档："It also means that a state proxy can be mutated in the child."

普通 prop 也可以变异（不推荐，会有 `ownership_invalid_mutation` 警告）。`$bindable` 是显式声明"我允许被改"。

## 与函数绑定的关系

`$bindable` 让 `bind:` 自动工作。若想自定义读写逻辑，用函数绑定：

```svelte
<script>
  let { value = $bindable(0), min = 0, max = 100 } = $props();

  function clamp(v: number) {
    return Math.max(min, Math.min(max, v));
  }
</script>

<input
  type="range"
  bind:value={() => value, (v) => (value = clamp(v))}
  min={min}
  max={max}
/>
```

## 完整示例：受控输入

```svelte
<!-- NumberField.svelte -->
<script lang="ts">
  let {
    value = $bindable(0),
    min = 0,
    max = 100,
    step = 1
  }: {
    value?: number;
    min?: number;
    max?: number;
    step?: number;
  } = $props();

  function clamp(v: number) {
    return Math.max(min, Math.min(max, v));
  }
</script>

<input
  type="number"
  value={value}
  oninput={(e) => (value = clamp(+e.currentTarget.value))}
  {min} {max} {step}
/>
```

```svelte
<!-- App.svelte -->
<script>
  let count = $state(0);
</script>
<NumberField bind:value={count} min={0} max={10} />
<p>current: {count}</p>
```

## 多个 bindable props

```svelte
<!-- DateRange.svelte -->
<script lang="ts">
  let {
    start = $bindable(''),
    end = $bindable('')
  }: { start?: string; end?: string } = $props();
</script>

<input type="date" bind:value={start} />
<input type="date" bind:value={end} />
```

```svelte
<DateRange bind:start bind:end />
```

## 与数组/对象共享（mutate 模式）

```svelte
<!-- TodoList.svelte -->
<script lang="ts">
  let { items = $bindable([]) }: {
    items: Array<{ id: number; text: string; done: boolean }>;
  } = $props();

  function add() {
    items.push({ id: Date.now(), text: '', done: false });
  }
</script>

<ul>
  {#each items as item (item.id)}
    <li>
      <input bind:value={item.text} />
      <input type="checkbox" bind:checked={item.done} />
    </li>
  {/each}
</ul>
<button onclick={add}>+</button>
```

父组件：

```svelte
<script>
  let todos = $state([{ id: 1, text: 'x', done: false }]);
</script>
<TodoList bind:items={todos} />
```

> 子组件 `items.push(...)` 会写回 `todos`（共享代理）。

## 性能 & 陷阱

1. **避免循环绑定**：父、子都 bind 同一个状态且都写它 → 死循环
2. **避免在大型对象上 bind** —— 每次深 mutation 都跨组件传播
3. **不要滥用** —— 数据流难以追踪。优先 callback props
4. **fallback 值的限制**：`$bindable([])` 的 `[]` 是**每次新数组**，每次组件实例化都不同

## $bindable 与 SSR

`bind:` 在 SSR 期间不工作（没有交互）。组件挂载到客户端后才开始响应 `bind:` 的双向更新。

## 类型化细节

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    value: string;
    onSubmit?: (v: string) => void;
    children?: Snippet;
  }

  let { value = $bindable(''), onSubmit, children }: Props = $props();

  function handleSubmit() {
    onSubmit?.(value);
  }
</script>

<form onsubmit={handleSubmit}>
  <input bind:value />
  {@render children?.()}
</form>
```

> `bindable<T>(value?: T)` 的返回类型是 `T`。`T` 默认为 `undefined`，所以未指定时类型推断为 `unknown` 或 `undefined`。要显式类型：声明 prop 接口。
