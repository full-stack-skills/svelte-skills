# style: / class / class: Reference

## style: 指令

为单个元素声明多个样式的简写。

```svelte
<!-- 等价 -->
<div style:color="red">...</div>
<div style="color: red;">...</div>
```

### 表达式

```svelte
<div style:color={myColor}>...</div>
<div style:color>...</div>  <!-- 简写 -->
```

### 多个

```svelte
<div
  style:color
  style:width="12rem"
  style:background-color={dark ? 'black' : 'white'}
>...</div>
```

### !important

```svelte
<div style:color|important="red">...</div>
```

> `style:` 指令优先级高于 `style` 属性 — **甚至**高于 `style` 中的 `!important`：

```svelte
<div style:color="red" style="color: blue">red</div>
<div style:color="red" style="color: blue !important">still red</div>
```

### CSS 自定义属性

```svelte
<div style:--columns={columns}>...</div>
```

可在子组件中用 `var(--columns, 4)` 读取。

## class 属性

### 字符串

```svelte
<div class={large ? 'large' : 'small'}>...</div>
```

> 历史行为：falsy 值（`false`、`NaN`）被序列化为字符串 `class="false"`；`undefined` / `null` 会**省略**属性。未来版本会改为所有 falsy 都省略。

### 对象（5.16+）

```svelte
<div class={{ cool, lame: !cool }}>...</div>
```

`cool` 为 truthy → `class="cool"`；否则 `class="lame"`。

### 数组（5.16+）

```svelte
<div class={[faded && 'saturate-0 opacity-50', large && 'scale-200']}>...</div>
```

falsy 值会被过滤。

### 嵌套

数组/对象可包含数组和对象，clsx 扁平化：

```svelte
<button class={['btn', { primary: true }, ['rounded', { large }]]}>
```

实际 `class="btn primary rounded large"`。

### ClassValue 类型（5.19+）

```svelte
<script lang="ts">
  import type { ClassValue } from 'svelte/elements';
  const props: { class: ClassValue } = $props();
</script>

<div class={['original', props.class]}>...</div>
```

## class: 指令（传统）

```svelte
<!-- 等价 -->
<div class={{ cool, lame: !cool }}>...</div>
<div class:cool={cool} class:lame={!cool}>...</div>

<!-- 简写 -->
<div class:cool class:lame={!cool}>...</div>
```

> 5.16+ 推荐用 class 属性对象/数组形式。

## 优先级

| 来源 | 优先级 |
|------|--------|
| `style:` 指令 + `!important` | 最高 |
| `style:` 指令 | 高（覆盖 `style` 属性 + `!important`） |
| `style` 属性 `!important` | 中 |
| `style` 属性 | 低 |
| scoped class (`svelte-xyz`) | 取决于 specificity |

`style:` 指令即使没有 `!important` 也覆盖 `style` 属性的 `!important`。

## scoped 样式注意事项

- `<style>` 默认 scoped：加 class 哈希（如 `svelte-abc`）
- specificity 提升 0-1-0
- `:global(...)` 关闭 scoping
- `style:` 指令注入的是内联样式 — 不经过 scoped 编译

## 实际模式

### 响应式 class 集合

```svelte
<script>
  let { variant = 'default', size = 'md' } = $props();
</script>

<button class={{
  btn: true,
  [`btn-${variant}`]: true,
  [`btn-${size}`]: true,
  'btn-disabled': disabled
}}>...</button>
```

### 组合 Tailwind

```svelte
<Button
  class={{ 'bg-blue-700 sm:w-1/2': useTailwind }}
  class:px-4={size === 'md'}
>...</Button>
```

### 通过 props 透传 class

```svelte
<button
  {...props}
  class={['btn', 'btn-primary', props.class]}
>...</button>
```

调用者：

```svelte
<Button class={{ 'bg-red-500': isError }}>...</Button>
```

### 状态切换

```svelte
<script>
  let isActive = $state(false);
</script>

<div
  class:is-active={isActive}
  class:opacity-50={!isActive}
  style:transform={isActive ? 'scale(1)' : 'scale(0.9)'}
>...</div>
```

## 性能提示

- `class` 对象/数组形式比 `class:` 指令**更快**（更少 DOM 操作）
- 避免在大型列表上用长 class 字符串拼接 — 用对象
- 静态类字符串用 `class="..."` 即可，动态用 `class={{}}`

## 错误对照

| 错误 | 原因 | 解决 |
|------|------|------|
| `class` 不应用 | falsy 值被序列化 | 用对象形式 |
| `style:` 不覆盖 | 用了 `style` 属性 | 改 `style:` 指令 |
| 自定义属性不生效 | 缺少 `--` 前缀 | `style:--var` |
| scoped 不应用 | 用了 `:global` 误关闭 | 检查 CSS |
