# $props 完整参考

## 基本解构

```svelte
<script lang="ts">
  let {
    name,
    age = 18,
    items = []
  }: {
    name: string;
    age?: number;
    items?: string[];
  } = $props();
</script>
```

> 编译器把 prop 访问转成 `get`/`set` 并追踪依赖，因此 prop 变化会自动重渲染。

## Fallback values

```js
let { adjective = 'happy' } = $props();
```

> 默认值仅在 prop 缺省或显式传 `undefined` 时生效。

> "Fallback values are not turned into reactive state proxies (see Updating props)"

## Renaming props

```js
let { super: trouper = 'lights are gonna find me' } = $props();
let { 'class': className, 'for': htmlFor } = $props();
```

> 适用：JS 关键字（`super`/`class`/`for` 等）、HTML 属性名与 JS 标识符冲突。

## Rest props

```svelte
<script lang="ts">
  let {
    variant = 'primary',
    size = 'md',
    ...rest
  }: {
    variant?: 'primary' | 'secondary';
    size?: 'sm' | 'md' | 'lg';
    [key: string]: any;
  } = $props();
</script>

<button class="btn btn-{variant} btn-{size}" {...rest}>
  <slot />
</button>
```

> 透传未消费的 props 到内部元素。配合 `svelte/elements` 的 `HTMLButtonAttributes` 等可强类型化。

## Updating props

### 临时覆盖（unsaved state）

子组件**可以临时**重新赋值 prop，父级不会受影响：

```svelte
<!-- App.svelte -->
<script>
  import Child from './Child.svelte';
  let count = $state(0);
</script>

<button onclick={() => (count += 1)}>parent: {count}</button>
<Child {count} />

<!-- Child.svelte -->
<script>
  let { count } = $props();
</script>

<button onclick={() => (count += 1)}>child: {count}</button>
```

> 父级重新渲染时子组件的临时值被覆盖。文档："the child component is able to temporarily override the prop value, which can be useful for unsaved ephemeral state."

### ❌ 不要 mutate props

```svelte
<!-- Child.svelte -->
<script>
  let { object } = $props();
</script>

<button onclick={() => (object.count += 1)}>
  <!-- 父组件里 object 是 $state({ count: 0 })：会更新但有 ownership_invalid_mutation warning -->
  <!-- 父组件里 object 是普通 { count: 0 }：不会更新 -->
  {object.count}
</button>
```

> 文档："don't mutate props. Either use callback props to communicate changes, or — if parent and child should share the same object — use the $bindable rune."

### Fallback 值不是代理

```svelte
<!-- Child.svelte -->
<script>
  let { object = { count: 0 } } = $props(); // 默认值不是代理
</script>

<button onclick={() => (object.count += 1)}>
  <!-- 父未传时：变更不触发更新 -->
  {object.count}
</button>
```

> 让默认值也可变：用 `$bindable({ count: 0 })`。

## Type safety

### TypeScript 内联注解

```svelte
<script lang="ts">
  let { adjective }: { adjective: string } = $props();
</script>
```

### JSDoc 风格

```svelte
<script>
  /** @type {{ adjective: string }} */
  let { adjective } = $props();
</script>
```

### 分离接口

```svelte
<script lang="ts">
  interface Props {
    adjective: string;
    count?: number;
  }

  let { adjective, count = 0 }: Props = $props();
</script>
```

> 文档："Interfaces for native DOM elements are provided in the svelte/elements module (see Typing wrapper components)"

## 包装原生元素

```svelte
<script lang="ts">
  import type { HTMLButtonAttributes } from 'svelte/elements';
  let { children, ...rest }: HTMLButtonAttributes = $props();
</script>

<button {...rest}>
  {@render children?.()}
</button>
```

> `svelte/elements` 提供 `HTMLButtonAttributes`、`HTMLInputAttributes`、`HTMLAnchorAttributes` 等。

## Snippet 作为 Props

```svelte
<!-- Modal.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte';
  let { title, children }: { title: string; children: Snippet } = $props();
</script>

<div class="modal">
  <h2>{title}</h2>
  {@render children()}
</div>
```

> Snippet 类型参数：`Snippet<[Arg1, Arg2]>` 表示带参数的 snippet。

## $props.id()（Svelte 5.20+）

生成**当前组件实例唯一**的 ID，SSR 时 server/client 一致：

```svelte
<script>
  const uid = $props.id();
</script>

<form>
  <label for="{uid}-firstname">First Name:</label>
  <input id="{uid}-firstname" type="text" />

  <label for="{uid}-lastname">Last Name:</label>
  <input id="{uid}-lastname" type="text" />
</form>
```

> 文档："This is useful for linking elements via attributes like for and aria-labelledby."

## 泛型组件

```svelte
<script lang="ts" generics="T extends { id: number }">
  import type { Snippet } from 'svelte';

  let {
    items,
    renderItem
  }: {
    items: T[];
    renderItem: Snippet<[T]>;
  } = $props();
</script>

{#each items as item (item.id)}
  {@render renderItem(item)}
{/each}
```

> `<script lang="ts" generics="...">` 声明泛型参数。Svelte 编译器会按调用点推断。

## Callback Props 模式

不依赖 `$bindable` 的双向通信：

```svelte
<!-- Toggle.svelte -->
<script lang="ts">
  let {
    value = $bindable(),
    onChange
  }: {
    value?: boolean;
    onChange?: (v: boolean) => void;
  } = $props();
</script>

<input
  type="checkbox"
  bind:checked={() => value, (v) => { value = v; onChange?.(v); }}
/>
```

> 函数绑定 `bind:checked={() => value, setter}` 让 `bind:` 同时支持读写自定义逻辑。

## 默认值规则

| 语法 | 行为 |
|------|------|
| `{ name }` | 必填 |
| `{ name = 'x' }` | 可选，默认 'x' |
| `{ name = $bindable() }` | 可选 + 双向绑定（fallback 是普通值） |
| `{ name = $bindable('x') }` | 可选 + 双向绑定，默认 'x' |
| `{ ...rest }` | 剩余 props |

## Prop 透传到根元素

`{...rest}` 是常见模式 —— 把未消费的 props 透传到根元素，避免手动列举：

```svelte
<script lang="ts">
  let { variant, ...rest }: { variant?: string; [k: string]: any } = $props();
</script>
<div {...rest} class="wrapper {variant}"><slot /></div>
```

## 解构 + 保留响应性

```svelte
<script>
  // ✅ 解构后变量仍响应（编译器处理）
  let { name, age } = $props();
  name = 'Bob'; // 临时覆盖
</script>
```

> 解构出的是 get/set 包装的局部变量，可读可写。
