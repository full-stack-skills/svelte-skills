# bind: Directives

## 1. bind:value

```svelte
<script>
  let name = $state('');
</script>

<input bind:value={name} />
<p>Hello, {name}!</p>
```

## 2. bind:checked（复选框）

```svelte
<script>
  let agreed = $state(false);
</script>

<label>
  <input type="checkbox" bind:checked={agreed} />
  I agree to the terms
</label>

{#if agreed}
  <button>Submit</button>
{/if}
```

## 3. bind:group（单选/多选）

```svelte
<script>
  let color = $state('red');
  let toppings = $state([]);
</script>

<!-- 单选 -->
<input type="radio" bind:group={color} value="red" /> Red
<input type="radio" bind:group={color} value="blue" /> Blue

<!-- 多选 -->
<input type="checkbox" bind:group={toppings} value="cheese" /> Cheese
<input type="checkbox" bind:group={toppings} value="onion" /> Onion
```

## 4. bind:files（文件输入）

```svelte
<script>
  let files = $state<FileList | null>(null);
</script>

<input type="file" bind:files multiple />
{#if files}
  <p>{files.length} files selected</p>
{/if}
```

## 5. bind:this（DOM 元素）

```svelte
<script>
  let canvas: HTMLCanvasElement;

  $effect(() => {
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    ctx.fillRect(0, 0, 100, 100);
  });
</script>

<canvas bind:this={canvas} width={200} height={200} />
```

## 6. bind:this（组件）

```svelte
<!-- Counter.svelte -->
<script lang="ts">
  class Counter {
    count = $state(0);
    increment() { this.count += 1; }
  }
  export const counter = new Counter();
</script>

<!-- Parent.svelte -->
<script>
  import Counter from './Counter.svelte';
  let counterRef;

  function handleClick() {
    counterRef.counter.increment();
  }
</script>

<Counter bind:this={counterRef} />
<button onclick={handleClick}>External +</button>
```

## 7. bind:value（contenteditable）

```svelte
<script>
  let html = $state('<p>Hello</p>');
</script>

<div contenteditable bind:textContent={html} />
<p>Raw: {html}</p>
```

## 8. bind:clientX/clientY（拖拽）

```svelte
<script>
  let x = $state(0);
  let y = $state(0);
</script>

<svelte:window
  onpointermove={(e) => {
    x = e.clientX;
    y = e.clientY;
  }}
/>

<div style="position: fixed; left: {x}px; top: {y}px;">
  Drag me
</div>
```

## 9. bind:offsetWidth/Height

```svelte
<script>
  let width = $state(0);
  let el: HTMLDivElement;
</script>

<div bind:offsetWidth={width}>
  Width: {width}px
</div>
```

## 10. bind:value + $bindable（双向）

```svelte
<!-- Input.svelte -->
<script lang="ts">
  let {
    value = $bindable('')
  }: { value?: string } = $props();
</script>

<input bind:value />

<!-- Parent.svelte -->
<script>
  let text = $state('hello');
</script>

<Input bind:value={text} />
```
