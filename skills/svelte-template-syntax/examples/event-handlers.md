# Event Handlers

## 1. onclick（推荐，Svelte 5）

```svelte
<script>
  let count = $state(0);
</script>

<button onclick={() => count += 1}>+1</button>
<p>{count}</p>
```

## 2. 行内箭头函数

```svelte
<button onclick={(e) => console.log('clicked', e.target)}>
  Click me
</button>
```

## 3. 事件修饰符（preventDefault 等）

```svelte
<script>
  function handleSubmit(e) {
    e.preventDefault();
    // ...
  }
</script>

<form onsubmit={handleSubmit}>
  <button>Submit</button>
</form>
```

## 4. 事件冒泡与 stopPropagation

```svelte
<div onclick={() => count += 1}>
  <button onclick={(e) => e.stopPropagation()}>
    Don't increment
  </button>
</div>
```

## 5. 事件委托（性能优化）

```svelte
<!-- 不推荐：为每个 item 绑定独立监听 -->
{#each items as item}
  <button onclick={() => handle(item.id)}>{item.name}</button>
{/each}

<!-- ✅ 推荐：事件委托到父元素 -->
<div onclick={(e) => {
  const btn = e.target.closest('button');
  if (btn) handle(Number(btn.dataset.id));
}}>
  {#each items as item}
    <button data-id={item.id}>{item.name}</button>
  {/each}
</div>
```

## 6. window/document 事件

```svelte
<svelte:window onkeydown={handleKey} />
<svelte:document onvisibilitychange={handleVisibility} />
<svelte:window onresize={() => console.log(innerWidth)} />
```

## 7. 事件对象类型

```svelte
<script lang="ts">
  function handleClick(e: MouseEvent) {
    console.log(e.clientX, e.clientY);
  }

  function handleInput(e: Event) {
    const target = e.target as HTMLInputElement;
    console.log(target.value);
  }
</script>

<button onclick={handleClick}>Click</button>
<input oninput={handleInput} />
```

## 8. 事件处理函数作为 Props

```svelte
<!-- Button.svelte -->
<script lang="ts">
  let {
    onclick
  }: { onclick?: (e: MouseEvent) => void } = $props();
</script>

<button {onclick}>
  <slot />
</button>
```

## 9. 键盘事件

```svelte
<script>
  function handleKey(e: KeyboardEvent) {
    if (e.key === 'Enter') {
      submit();
    }
  }
</script>

<input
  onkeydown={handleKey}
  placeholder="Press Enter to submit"
/>
```

## 10. 指针事件（统一鼠标/触摸）

```svelte
<div
  onpointerdown={(e) => console.log('pointer down', e.pointerId)}
  onpointermove={(e) => console.log('moved to', e.clientX, e.clientY)}
  onpointerup={() => console.log('pointer up')}
>
  Drag here
</div>
```

## 11. 旧语法 on:click（Svelte 4 兼容）

```svelte
<!-- Svelte 5 推荐 onclick，on:click 仍可用但已废弃 -->
<button on:click={handleClick}>Legacy</button>
```

## 12. 动态事件名

```svelte
<script>
  let eventName = 'click';
</script>

<!-- 动态事件名 -->
<button
  onclick={() => console.log('clicked')}
  onkeydown={(e) => e.key === 'Enter' && console.log('enter')}
>
  Click or Enter
</button>
```
