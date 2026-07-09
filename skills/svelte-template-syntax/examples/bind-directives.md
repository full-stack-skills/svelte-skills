# bind: Directives

`bind:` 允许数据从子级回流到父级（默认是单向向下）。可读可写（双向）或只读。

## 1. bind:value（文本输入）

```svelte
<script>
  let name = $state('');
</script>

<input bind:value={name} />
<p>Hello, {name}!</p>
```

## 2. bind:value（数字输入）

`type="number"` / `type="range"` 自动转 number；空值或无效值为 `undefined`：

```svelte
<script>
  let a = $state(1);
</script>

<input type="number" bind:value={a} min="0" max="10" />
<input type="range" bind:value={a} min="0" max="10" />
```

## 3. bind:value（textarea）

```svelte
<textarea bind:value></textarea>
```

## 4. bind:value（select）

```svelte
<script>
  let selected = $state('a');
</script>

<select bind:value={selected}>
  <option value="a">A</option>
  <option value="b">B</option>
  <option value="c">C</option>
</select>
```

`<option value>` 是属性，可以是任意类型（不限于字符串）。

### 多选 select

```svelte
<script>
  let fillings = $state([]);
</script>

<select multiple bind:value={fillings}>
  <option>Rice</option>
  <option>Beans</option>
  <option>Cheese</option>
</select>
```

## 5. bind:checked（复选框）

```svelte
<script>
  let agreed = $state(false);
</script>

<label>
  <input type="checkbox" bind:checked={agreed} />
  I agree
</label>
```

> 单选用 `bind:group`，不要用 `bind:checked`。

## 6. bind:indeterminate

```svelte
<script>
  let checked = $state(false);
  let indeterminate = $state(true);
</script>

<input type="checkbox" bind:checked bind:indeterminate />
```

## 7. bind:group（单选）

```svelte
<script>
  let color = $state('red');
</script>

<input type="radio" bind:group={color} value="red" /> Red
<input type="radio" bind:group={color} value="blue" /> Blue
```

## 8. bind:group（多选）

```svelte
<script>
  let toppings = $state([]);
</script>

<input type="checkbox" bind:group={toppings} value="cheese" /> Cheese
<input type="checkbox" bind:group={toppings} value="onion" /> Onion
```

> `bind:group` 仅在同一 Svelte 组件内有效。

## 9. bind:files（文件输入）

```svelte
<script>
  let files = $state();
</script>

<input type="file" bind:files multiple accept="image/png, image/jpeg" />

<!-- 清空（FileList 不可直接构造） -->
<script>
  function clear() {
    files = new DataTransfer().files;
  }
</script>
<button onclick={clear}>clear</button>
```

> `DataTransfer` 在 SSR 不可用 — 文件绑定状态保持未初始化以避免错误。

## 10. bind:open（details）

```svelte
<script>
  let isOpen = $state(false);
</script>

<details bind:open={isOpen}>
  <summary>How do you comfort a JavaScript bug?</summary>
  <p>You console it.</p>
</details>
```

## 11. contenteditable 绑定

三种：`bind:innerHTML` / `bind:innerText` / `bind:textContent`：

```svelte
<script>
  let html = $state('<p>Hello</p>');
  let text = $state('Hello');
</script>

<div contenteditable="true" bind:innerHTML={html}></div>
<div contenteditable="true" bind:textContent={text}></div>
```

> `innerText` 与 `textContent` 行为不同 — 详见 MDN。

## 12. 尺寸绑定（readonly，ResizeObserver）

适用：所有可见元素（`display: inline` 元素除外，除非有内禀尺寸如 `<img>` / `<canvas>`）。

```svelte
<script>
  let width = $state(0);
  let height = $state(0);
</script>

<div bind:offsetWidth={width} bind:offsetHeight={height}>
  <Chart {width} {height} />
</div>
```

可用绑定：
- `clientWidth` / `clientHeight`
- `offsetWidth` / `offsetHeight`
- `contentRect`
- `contentBoxSize` / `borderBoxSize` / `devicePixelContentBoxSize`

## 13. <audio> / <video> 绑定

`bind:duration` / `bind:currentTime` / `bind:paused` / `bind:volume` / `bind:muted` / `bind:playbackRate`（双向）

readonly：`buffered` / `seekable` / `seeking` / `ended` / `readyState` / `played`

`<video>` 额外：`videoWidth` / `videoHeight`（readonly）

```svelte
<audio
  src={clip}
  bind:duration
  bind:currentTime
  bind:paused
  bind:volume
  bind:muted
></audio>

<video
  src={clip}
  bind:currentTime
  bind:videoWidth
  bind:videoHeight
></video>
```

## 14. <img> 绑定

`bind:naturalWidth` / `bind:naturalHeight`（只读）

```svelte
<img src={url} bind:naturalWidth bind:naturalHeight />
```

## 15. bind:this（DOM 引用）

```svelte
<script>
  /** @type {HTMLCanvasElement} */
  let canvas;

  $effect(() => {
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    drawStuff(ctx);
  });
</script>

<canvas bind:this={canvas}></canvas>
```

> 组件挂载前值是 `undefined`，在 effect 或事件处理器中读取。

## 16. bind:this（组件实例）

```svelte
<!-- App.svelte -->
<ShoppingCart bind:this={cart} />
<button onclick={() => cart.empty()}>Empty</button>

<!-- ShoppingCart.svelte -->
<script>
  export function empty() {
    // ...
  }
</script>
```

## 17. Function bindings（5.9+）

用 getter/setter 函数进行校验/转换：

```svelte
<input bind:value={
  () => value,
  (v) => value = v.toLowerCase()
} />
```

readonly 绑定（如尺寸）用 `null` 占位 getter：

```svelte
<div
  bind:clientWidth={null, redraw}
  bind:clientHeight={null, redraw}
>...</div>
```

> 组件/元素销毁时，getter 用于正确清空值。

## 18. bindable props（$bindable）

子组件中允许父级绑定：

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

带 fallback：

```svelte
<script>
  let { value = $bindable('fallback value') } = $props();
</script>
```

> 当属性被绑定且有 fallback 时，父级必须提供非 `undefined` 值，否则抛运行时错误。

## 19. form reset 行为

5.6+ 起带 `defaultValue` / `defaultChecked` 的输入在 form reset 时恢复：

```svelte
<form>
  <input bind:value defaultValue="not the empty string" />
  <input type="checkbox" bind:checked defaultChecked={true} />
  <input type="reset" value="Reset">
</form>
```

> 初始渲染时 binding 优先（除非 `null` / `undefined`）。

## 20. bind: 与事件

绑定对应的 `oninput` / `onchange` 事件仍可同时使用 — 事件处理器**先**于绑定更新触发。
