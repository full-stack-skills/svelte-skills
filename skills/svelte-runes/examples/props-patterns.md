# $props and $bindable Patterns

## 1. 基础 $props

```svelte
<!-- Child.svelte -->
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

<p>{name}, {age}</p>
<ul>
  {#each items as item}
    <li>{item}</li>
  {/each}
</ul>

<!-- Parent.svelte -->
<Child name="Alice" age={25} items={['a', 'b']} />
<Child name="Bob" />  <!-- age=18, items=[] -->
```

> 默认值仅在 prop 缺省/`undefined` 时生效；显式传 `null` 不算缺省。

## 2. $props + $derived

```svelte
<script lang="ts">
  let {
    price,
    quantity = 1,
    discount = 0
  }: {
    price: number;
    quantity?: number;
    discount?: number;
  } = $props();

  let subtotal = $derived(price * quantity);
  let total = $derived(subtotal * (1 - discount));
</script>
```

## 3. $bindable 可变绑定

```svelte
<!-- Input.svelte -->
<script lang="ts">
  let {
    value = $bindable(''),
    placeholder = 'Enter text...'
  }: {
    value?: string;
    placeholder?: string;
  } = $props();
</script>

<input bind:value {placeholder} />

<!-- Parent.svelte -->
<script>
  let text = $state('hello');
</script>

<!-- 使用 bind: -->
<Input bind:value={text} />
<!-- 或不用 bind（单向）：-->
<Input value={text} />
<p>你输入了: {text}</p>
```

## 4. $bindable 乐观更新

```svelte
<!-- TodoItem.svelte -->
<script lang="ts">
  let {
    todo,
    onToggle = () => {}
  }: {
    todo: { id: number; text: string; done: boolean };
    onToggle?: (id: number) => void;
  } = $props();

  let { done = $bindable(todo.done) } = $props();
</script>

<input
  type="checkbox"
  bind:checked={done}
  onchange={() => onToggle(todo.id)}
/>
<span class:done>{todo.text}</span>

<!-- Parent.svelte -->
<script>
  let todos = $state([
    { id: 1, text: 'Learn Svelte', done: false },
    { id: 2, text: 'Build an app', done: false }
  ]);

  async function handleToggle(id) {
    // 乐观 UI：done 已在子组件更新
    await fetch(`/api/toggle/${id}`, { method: 'POST' });
  }
</script>

{#each todos as todo}
  <TodoItem {todo} onToggle={handleToggle} />
{/each}
```

## 5. Rest Props 透传

```svelte
<script lang="ts">
  let {
    variant = 'primary',
    size = 'md',
    ...rest
  }: {
    variant?: 'primary' | 'secondary' | 'danger';
    size?: 'sm' | 'md' | 'lg';
    [key: string]: any;
  } = $props();
</script>

<button
  class="btn btn-{variant} btn-{size}"
  {...rest}    <!-- 透传所有剩余 props -->
>
  <slot />
</button>

<!-- 使用 -->
<Button variant="danger" size="sm" onclick={() => handleDelete(id)}>
  删除
</Button>
```

> Rest props 的类型建议用 `[key: string]: any` 或 `HTMLAttributes<HTMLButtonElement>` 约束。

## 6. $props + slots/snippets

```svelte
<!-- Modal.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte';

  let {
    title,
    children
  }: {
    title: string;
    children: Snippet;
  } = $props();
</script>

<div class="modal">
  <h2>{title}</h2>
  {@render children()}
</div>

<!-- Parent.svelte -->
<Modal title="Welcome">
  {#snippet children()}
    <p>Hello, world!</p>
  {/snippet}
</Modal>
```

## 7. 类型安全的 $props（泛型组件）

```svelte
<script lang="ts" generics="T extends string | number">
  import type { Snippet } from 'svelte';

  let {
    items,
    renderItem
  }: {
    items: T[];
    renderItem: Snippet<[T]>;
  } = $props();
</script>

{#each items as item}
  {@render renderItem(item)}
{/each>

<!-- 使用 -->
<List items={['a', 'b', 'c']}>
  {#snippet renderItem(item: string)}
    <span>{item}</span>
  {/snippet}
</List>
```

## 8. 临时覆盖 prop（unsaved 状态）

子组件**可以临时重新赋值** prop：

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

父组件的 `count` 不会受子组件修改影响；父组件重新渲染时子组件的临时值会被覆盖。

## 9. prop 重命名

```svelte
<script lang="ts">
  let {
    'class': className,
    'for': htmlFor,
    super: trouper = 'default'
  }: {
    'class'?: string;
    'for'?: string;
    super?: string;
  } = $props();
</script>

<label for={htmlFor} class={className}>{trouper}</label>
```

## 10. 默认值不是响应式代理

```svelte
<!-- Child.svelte -->
<script>
  let { object = { count: 0 } } = $props();
</script>

<button onclick={() => (object.count += 1)}>
  clicks: {object.count}
</button>
```

如果父组件**未传** `object`，点击不会触发更新（默认值不是代理）。要让默认也可变：用 `$bindable({ count: 0 })`。

## 11. 错误：变异 prop

```svelte
<!-- Child.svelte -->
<script>
  let { object } = $props();
</script>

<button onclick={() => (object.count += 1)}>
  <!-- 父组件里 object 是 $state({ count: 0 }) → 会更新但有 warning -->
  <!-- 父组件里 object 是普通 { count: 0 } → 不会更新 -->
  {object.count}
</button>
```

> 控制台会出现 `ownership_invalid_mutation` 警告。正确做法：callback props 或 `$bindable`。

## 12. 类型安全 —— interface + 解构注解

```svelte
<script lang="ts">
  interface Props {
    adjective: string;
    count?: number;
  }

  let { adjective, count = 0 }: Props = $props();
</script>
```

JSDoc 风格（无 TS 时）：

```svelte
<script>
  /** @type {{ adjective: string, count?: number }} */
  let { adjective, count = 0 } = $props();
</script>
```

## 13. 包装原生元素

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { HTMLButtonAttributes } from 'svelte/elements';

  let { children, ...rest }: HTMLButtonAttributes = $props();
</script>

<button {...rest}>
  {@render children?.()}
</button>
```

> `svelte/elements` 提供 `HTMLButtonAttributes`、`HTMLInputAttributes` 等原生元素属性类型。

## 14. $props.id() —— 唯一 ID

Svelte 5.20+ 提供 `$props.id()`，生成**当前组件实例唯一**的 ID，SSR 时 server/client 一致：

```svelte
<script>
  const uid = $props.id();
</script>

<form>
  <label for="{uid}-firstname">First Name: </label>
  <input id="{uid}-firstname" type="text" />

  <label for="{uid}-lastname">Last Name: </label>
  <input id="{uid}-lastname" type="text" />
</form>
```

> 适合 `for`/`aria-labelledby`/`aria-describedby` 等需要 ID 配对的场景。

## 15. Callback Props 模式

不依赖 `$bindable` 的双向通信：

```svelte
<!-- Toggle.svelte -->
<script lang="ts">
  let { onChange, label }: { onChange: (v: boolean) => void; label: string } = $props();
  let value = $state(false);
</script>

<label>
  <input type="checkbox" bind:checked={() => value, (v) => { value = v; onChange(v); }} />
  {label}
</label>

<!-- Parent.svelte -->
<Toggle label="Enable" onChange={(v) => console.log('changed to', v)} />
```

## 16. 类型化 snippet 参数

```svelte
<script lang="ts" generics="T">
  import type { Snippet } from 'svelte';

  let {
    items,
    row
  }: {
    items: T[];
    row: Snippet<[T, number]>;   // (item, index)
  } = $props();
</script>

{#each items as item, i}
  {@render row(item, i)}
{/each}
```

```svelte
<!-- 用法 -->
<Table items={users}>
  {#snippet row(user, i)}
    <tr><td>{i + 1}</td><td>{user.name}</td></tr>
  {/snippet}
</Table>
```
