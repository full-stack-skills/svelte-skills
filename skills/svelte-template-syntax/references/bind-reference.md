# bind: Directives 完整参考

## 绑定类型表

| 指令 | 适用元素 | 绑定内容 |
|------|---------|---------|
| `bind:value` | input, textarea, select | 文本值 |
| `bind:checked` | input[type=checkbox/radio] | 布尔值 |
| `bind:group` | 多个 input | 单选/多选值 |
| `bind:files` | input[type=file] | FileList |
| `bind:this` | 任何元素/组件 | DOM 引用 |
| `bind:clientX/Y` | svelte:window | 指针 X/Y |
| `bind:offsetWidth/Height` | 块级元素 | 尺寸 |
| `bind:textContent` | contenteditable | HTML 内容 |
| `bind:innerHTML` | contenteditable | 纯 HTML |
| `bind:open` | details | open 状态 |
| `bind:volume` | audio/video | 音量 |
| `bind:muted` | audio/video | 静音 |
| `bind:paused` | audio/video | 暂停状态 |
| `bind:playbackRate` | audio/video | 播放速率 |

## bind:value 细节

```svelte
<script>
  let value = $state('');
</script>

<!-- 输入框 -->
<input bind:value />

<!-- textarea -->
<textarea bind:value></textarea>

<!-- select -->
<select bind:value>
  <option value="a">A</option>
  <option value="b">B</option>
</select>
```

## bind:checked

```svelte
<script>
  let agree = $state(false);
</script>

<input type="checkbox" bind:checked={agree} />
```

## bind:group

```svelte
<script>
  let choice = $state('a');
  let multi = $state([]);
</script>

<!-- 单选 -->
<input type="radio" bind:group={choice} value="a" />
<input type="radio" bind:group={choice} value="b" />

<!-- 多选 -->
<input type="checkbox" bind:group={multi} value="x" />
<input type="checkbox" bind:group={multi} value="y" />
```

## bind:this

```svelte
<script>
  let canvas: HTMLCanvasElement;

  $effect(() => {
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    // draw...
  });
</script>

<canvas bind:this={canvas} width={200} height={200} />
```

## bind:clientX/Y

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

<div style="position:fixed; left:{x}px; top:{y}px;">({x}, {y})</div>
```

## $bindable（双向 Props）

```svelte
<!-- Child.svelte -->
<script>
  let { value = $bindable() } = $props();
</script>
<input bind:value />

<!-- Parent.svelte -->
<script>
  let text = $state('hello');
</script>

<Child bind:value={text} />
```
