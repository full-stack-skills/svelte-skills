# $bindable Patterns

> 双向数据流：让 prop 可以**反向**从子组件写到父组件。谨慎使用 —— 数据流难以追踪。

## 1. 基础双向绑定

```svelte
<!-- FancyInput.svelte -->
<script>
  let { value = $bindable(), ...props } = $props();
</script>

<input bind:value={value} {...props} />

<style>
  input {
    font-family: 'Comic Sans MS';
    color: deeppink;
  }
</style>
```

```svelte
<!-- App.svelte -->
<script>
  import FancyInput from './FancyInput.svelte';
  let message = $state('hello');
</script>

<FancyInput bind:value={message} />
<p>{message}</p>
```

## 2. 带 fallback 的 $bindable

父组件**不**使用 `bind:` 时，`$bindable('fallback')` 作为默认值：

```js
// FancyInput.svelte
let { value = $bindable('fallback') } = $props();
```

```svelte
<!-- 单向：父不监听子 -->
<FancyInput value={parentValue} />

<!-- 双向：父监听子 -->
<FancyInput bind:value={parentValue} />
```

## 3. 透传所有 props + bind value

```svelte
<!-- TextField.svelte -->
<script lang="ts">
  let { value = $bindable(''), ...rest }: { value?: string; [k: string]: any } = $props();
</script>

<input bind:value {...rest} />
```

## 4. 与 $state 对象字段共享（不推荐替代品）

```svelte
<!-- FieldEditor.svelte -->
<script lang="ts">
  let { item = $bindable() }: { item: { name: string; value: string } } = $props();
</script>

<input bind:value={item.name} />
<input bind:value={item.value} />
```

```svelte
<script>
  let config = $state({ name: '', value: '' });
</script>
<FieldEditor bind:item={config} />
```

> 子组件修改 `item.name` 会写回 `config.name`（共享代理）。

## 5. 配合函数绑定的非受控变体

```svelte
<!-- Slider.svelte -->
<script lang="ts">
  let { value = $bindable(0) }: { value?: number } = $props();

  // 内部也可以重置
  function reset() { value = 0; }
</script>

<input type="range" bind:value />
<button onclick={reset}>Reset</button>
```

## 6. 数字输入：限制范围

```svelte
<!-- NumberInput.svelte -->
<script lang="ts">
  let { value = $bindable(0), min = 0, max = 100 }: {
    value?: number;
    min?: number;
    max?: number;
  } = $props();

  function clamp(v: number) {
    return Math.max(min, Math.min(max, v));
  }
</script>

<input
  type="number"
  value={value}
  oninput={(e) => (value = clamp(+e.currentTarget.value))}
/>
```

## 7. 复选框（boolean）

```svelte
<!-- Checkbox.svelte -->
<script lang="ts">
  let { checked = $bindable(false), label }: { checked?: boolean; label: string } = $props();
</script>

<label>
  <input type="checkbox" bind:checked />
  {label}
</label>
```

## 8. Select 绑定对象/字符串

```svelte
<!-- ColorPicker.svelte -->
<script lang="ts">
  let { color = $bindable('#ff3e00') }: { color?: string } = $props();
</script>

<input type="color" bind:value={color} />
<p>Selected: {color}</p>
```

## 9. 多个 bindable props

```svelte
<!-- DateRange.svelte -->
<script lang="ts">
  let { start = $bindable(''), end = $bindable('') }: {
    start?: string;
    end?: string;
  } = $props();
</script>

<input type="date" bind:value={start} />
<input type="date" bind:value={end} />
```

```svelte
<DateRange bind:start bind:end />
```

## 10. 数组方法修改 bindable 对象

```svelte
<!-- TodoList.svelte -->
<script lang="ts">
  let { items = $bindable([]) }: { items: Array<{ id: number; text: string; done: boolean }> } = $props();

  function add(text: string) {
    items.push({ id: Date.now(), text, done: false });
  }
</script>
```

## 11. 验证 / 规范化写入

```svelte
<!-- EmailInput.svelte -->
<script lang="ts">
  let { value = $bindable('') }: { value?: string } = $props();

  function onInput(e: Event) {
    const t = (e.currentTarget as HTMLInputElement).value;
    value = t.trim().toLowerCase(); // 写回前规范化
  }
</script>

<input {value} oninput={onInput} type="email" />
```

## 12. 何时避免 $bindable

| 场景 | 推荐 |
|------|------|
| 父→子单向 | 普通 props |
| 父监听子事件 | callback props |
| 子需要受控但无需重置 | `$bindable`（仅必要的最小集合） |
| 复杂状态机 | 用 Context / store / 状态机库 |

> 文档原话："This isn't something you should do often — overuse can make your data flow unpredictable and your components harder to maintain."

## 13. 在 bindable 内暴露子组件的局部状态

```svelte
<!-- Tabs.svelte -->
<script lang="ts">
  let { active = $bindable(0) }: { active?: number } = $props();
  const tabs = ['Home', 'Profile', 'Settings'];
</script>

<div class="tabs">
  {#each tabs as t, i}
    <button class:active={i === active} onclick={() => (active = i)}>{t}</button>
  {/each}
</div>

<div class="panel">{tabs[active]}</div>
```

```svelte
<Tabs bind:active={currentTab} />
```

## 14. 与 $inspect 配合调试

```svelte
<script>
  let { value = $bindable('') } = $props();
  $inspect(value); // 看子→父流
</script>
```

> $inspect 在生产构建中是 noop。
