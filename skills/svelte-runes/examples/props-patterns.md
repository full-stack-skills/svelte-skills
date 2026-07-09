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

## 6. $props + slots/snippets

```svelte
<!-- Modal.svelte -->
<script lang="ts">
  let {
    title,
    children
  }: {
    title: string;
    children: import('svelte').Snippet;
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
{/each}

<!-- 使用 -->
<List items={['a', 'b', 'c']}>
  {#snippet renderItem(item: string)}
    <span>{item}</span>
  {/snippet}
</List>
```
