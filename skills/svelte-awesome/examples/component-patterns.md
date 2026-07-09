# Svelte Component Patterns Examples

## 1. Props 传递与类型

### 基础 Props

```svelte
<script lang="ts">
  let {
    name,
    count = 0,
    onIncrement
  }: {
    name: string;
    count?: number;
    onIncrement: () => void;
  } = $props();
</script>

<p>Hello {name}, count: {count}</p>
<button onclick={onIncrement}>+</button>
```

### $bindable 可变绑定

```svelte
<!-- FancyInput.svelte -->
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
```

```svelte
<!-- 父组件 -->
<script lang="ts">
  let text = $state('hello');
</script>

<FancyInput bind:value={text} />
<p>输入了: {text}</p>
```

### Rest Props 透传

```svelte
<script lang="ts">
  let { count = 0, ...others }: { count?: number; [key: string]: any } = $props();
</script>

<p>count: {count}</p>
<button {...others}>Button</button>
```

## 2. Snippet 组件复用

### 卡片 Snippet

```svelte
<script lang="ts">
  let { items }: { items: Array<{id: number; title: string; body: string}> = $props();

  function snippet card(item: typeof items[0]) {
    return html`
      <div class="card">
        <h3>${item.title}</h3>
        <p>${item.body}</p>
      </div>
    `;
  }
</script>
```

### 动态 Snippet 条件渲染

```svelte
<script lang="ts">
  let { isLoggedIn, children, fallback }: {
    isLoggedIn: boolean;
    children: import('svelte').Snippet;
    fallback?: import('svelte').Snippet;
  } = $props();
</script>

{#if isLoggedIn}
  {@render children()}
{:else if fallback}
  {@render fallback()}
{/if}
```

### 递归 Snippet（树形结构）

```svelte
<script lang="ts">
  let {
    node,
    children
  }: {
    node: { label: string; children?: typeof node[] };
    children: import('svelte').Snippet<[typeof node]>;
  } = $props();
</script>

<li>
  <span>{node.label}</span>
  {#if node.children?.length}
    <ul>
      {#each node.children as child}
        <li>
          {@render children(child)}
        </li>
      {/each}
    </ul>
  {/if}
</li>
```

## 3. 事件处理

### 事件属性语法（onclick 而非 on:click）

```svelte
<script lang="ts">
  function handleSubmit(e: SubmitEvent) {
    e.preventDefault();
    const form = e.currentTarget;
    const data = new FormData(form);
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter') handleSubmit(e as unknown as SubmitEvent);
  }
</script>

<form onsubmit={handleSubmit}>
  <input name="email" onkeydown={handleKeydown} />
  <button type="submit">Submit</button>
</form>
```

### Event delegation（自动，无需手动 stopPropagation）

```svelte
<!-- 事件自动委托到根节点，stopPropagation 不会生效 -->
<!-- 如需阻止，用 svelte/events 的 on 函数 -->
<script>
  import { on } from 'svelte/events';
</script>

<div use:on={(node, event) => {
  // 正确处理委托和 stopPropagation
  // ...
}}>
```

## 4. 双向绑定

### bind:value / bind:checked

```svelte
<script lang="ts">
  let name = $state('');
  let agreed = $state(false);
  let lang = $state('zh');
</script>

<input bind:value={name} placeholder="Your name" />
<input type="checkbox" bind:checked={agreed} />
<select bind:value={lang}>
  <option value="zh">中文</option>
  <option value="en">English</option>
</select>
```

### bind:group（Radio / Checkbox）

```svelte
<script lang="ts">
  let plan = $state('pro');
  let features = $state<string[]>([]);
</script>

<input type="radio" bind:group={plan} value="free" />
<input type="radio" bind:group={plan} value="pro" />
<input type="checkbox" bind:group={features} value="analytics" />
<input type="checkbox" bind:group={features} value="support" />
```

## 5. 响应式派生

```svelte
<script lang="ts">
  let items = $state([
    { name: 'Apple', price: 3, qty: 2 },
    { name: 'Banana', price: 1, qty: 5 }
  ]);

  // 派生计算
  let total = $derived(
    items.reduce((sum, item) => sum + item.price * item.qty, 0)
  );

  let expensive = $derived(items.filter(i => i.price > 2));
</script>
```

## 6. 副作用与清理

```svelte
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';

  let canvas: HTMLCanvasElement;

  onMount(() => {
    const ctx = canvas.getContext('2d');
    // ...
    return () => { /* cleanup */ };
  });

  // 或用 $effect
  let width = $state(100);

  $effect(() => {
    canvas.width = width; // 追踪 width
  });
</script>

<canvas bind:this={canvas} width={width} height={100} />
```

## 7. 跨组件状态

### Context（层级传递）

```ts
// context.ts
import { createContext } from 'svelte';
export const [getUser, setUser] = createContext<{ name: string; age: number }>();
```

```svelte
<!-- Parent.svelte -->
<script lang="ts">
  import { setUser } from './context';
  setUser({ name: 'Alice', age: 30 });
</script>
{@render children()}

<!-- Child.svelte -->
<script lang="ts">
  import { getUser } from './context';
  const user = getUser(); // { name: 'Alice', age: 30 }
</script>
```

### 全局 .svelte.js 共享状态

```js
// state/user.svelte.js
export const userState = $state({
  name: '',
  email: ''
});
```

```svelte
<!-- 任意组件直接导入 -->
<script lang="ts">
  import { userState } from '$lib/state/user.svelte.js';
</script>
<p>{$userState.name}</p>
```
