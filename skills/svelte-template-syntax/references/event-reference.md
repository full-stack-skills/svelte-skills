# Event Handlers 完整参考

## Svelte 5 推荐：onclick

```svelte
<script>
  function handleClick() { /* ... */ }
</script>

<button onclick={handleClick}>Click</button>

<!-- 行内 -->
<button onclick={() => count += 1}>+1</button>
```

## 事件修饰符

Svelte 5 中不再有 `onclick|preventDefault` 语法，在处理器中调用：

```svelte
<form onsubmit={(e) => { e.preventDefault(); handleSubmit(); }}>
  <button>Submit</button>
</form>
```

## svelte:window 事件

```svelte
<svelte:window onkeydown={handleKey} onresize={handleResize} />
<svelte:document onvisibilitychange={handleVisibility} />
<svelte:window onbeforeprint={() => isPrinting = true} />
```

## on:click（旧语法，废弃但可用）

```svelte
<!-- Svelte 4 语法，仍支持但不推荐 -->
<button on:click={handleClick}>Legacy</button>
```

## 事件冒泡控制

```svelte
<div onclick={() => count += 1}>
  <!-- 阻止冒泡 -->
  <button onclick={(e) => e.stopPropagation()}>
    No bubble
  </button>
</div>
```

## 事件委托

```svelte
<!-- 推荐：单事件监听器处理多个子元素 -->
<ul onclick={(e) => {
  const li = e.target.closest('li[data-id]');
  if (li) handleSelect(Number(li.dataset.id));
}}>
  {#each items as item}
    <li data-id={item.id}>{item.name}</li>
  {/each}
</ul>
```

## 事件类型标注

```svelte
<script lang="ts">
  function handleClick(e: MouseEvent) {
    console.log(e.clientX, e.clientY);
  }

  function handleInput(e: Event) {
    const target = e.target as HTMLInputElement;
    console.log(target.value);
  }

  function handleKey(e: KeyboardEvent) {
    if (e.key === 'Enter') submit();
  }
</script>

<button onclick={handleClick}>Click</button>
<input oninput={handleInput} />
<input onkeydown={handleKey} />
```

## 动态事件处理

```svelte
<script>
  let handler: ((e: MouseEvent) => void) | null = null;

  function enable() {
    handler = () => console.log('enabled!');
  }
  function disable() {
    handler = null;
  }
</script>

<button {onclick}={handler}>Action</button>
```
