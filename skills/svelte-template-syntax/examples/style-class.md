# style: / class: Directives

Svelte 提供 `style:` 和 `class:` 指令，代替长的 `style` / `class` 属性。

## 1. style: 基础

```svelte
<!-- 等价 -->
<div style:color="red">...</div>
<div style="color: red;">...</div>
```

## 2. style: 表达式

```svelte
<div style:color={myColor}>...</div>

<!-- 简写：name={name} -->
<div style:color>...</div>
```

## 3. style: 多个

```svelte
<div
  style:color
  style:width="12rem"
  style:background-color={darkMode ? 'black' : 'white'}
>...</div>
```

## 4. style:|important

```svelte
<div style:color|important="red">...</div>
```

> `style:` 指令优先级高于 `style` 属性，**甚至**高于 `style` 中的 `!important`。

```svelte
<!-- 这会是红色 -->
<div style:color="red" style="color: blue">This will be red</div>
<div style:color="red" style="color: blue !important">This will still be red</div>
```

## 5. CSS 自定义属性

```svelte
<div style:--columns={columns}>...</div>
```

可在子组件中读取：`var(--columns, 4)`。

## 6. class 属性（字符串）

```svelte
<div class={large ? 'large' : 'small'}>...</div>
```

> 历史原因：falsy 值（`false`、`NaN`）被序列化为字符串 `class="false"`，但 `class={undefined}` / `null` 会**省略**属性。

## 7. class 属性（对象/数组，Svelte 5.16+）

```svelte
<script>
  let { cool } = $props();
</script>

<!-- 对象：truthy 的 key 添加 -->
<div class={{ cool, lame: !cool }}>...</div>

<!-- 数组：truthy 值合并 -->
<div class={[faded && 'saturate-0 opacity-50', large && 'scale-200']}>...</div>
```

数组/对象内部用 [clsx](https://github.com/lukeed/clsx) 扁平化：

```svelte
<button {...props} class={['cool-button', props.class]}>
  {@render props.children?.()}
</button>
```

调用者可以混用：

```svelte
<Button
  onclick={() => useTailwind = true}
  class={{ 'bg-blue-700 sm:w-1/2': useTailwind }}
>
  Accept Tailwind
</Button>
```

## 8. ClassValue 类型

```svelte
<script lang="ts">
  import type { ClassValue } from 'svelte/elements';

  const props: { class: ClassValue } = $props();
</script>

<div class={['original', props.class]}>...</div>
```

## 9. class: 指令（传统）

> 5.16+ 推荐用 class 对象/数组形式，`class:` 仍可用但稍弱。

```svelte
<!-- 等价 -->
<div class={{ cool, lame: !cool }}>...</div>
<div class:cool={cool} class:lame={!cool}>...</div>

<!-- 简写：name={name} -->
<div class:cool class:lame={!cool}>...</div>
```

## 10. 实际示例：响应式样式

```svelte
<script>
  let dark = $state(false);
  let width = $state(100);
  let color = $state('blue');
</script>

<div
  class:dark
  class:light={!dark}
  style:width="{width}px"
  style:color
>content</div>
```

## 11. 条件类集合（Tailwind）

```svelte
<script>
  let isActive = $state(false);
  let isError = $state(false);
</script>

<div class={{
  'px-4 py-2 rounded': true,           // 总是添加
  'bg-blue-500 text-white': isActive,  // 条件
  'border-red-500 ring-2': isError    // 条件
}}>...</div>
```

## 12. 嵌套数组/对象

```svelte
<script>
  let base = ['btn', 'rounded'];
  let states = { primary: true, disabled: false };
</script>

<button class={[...base, states, 'text-lg']}>...</button>
<!-- class="btn rounded primary text-lg" -->
```

## 13. 切换根级 class

```svelte
<script>
  let { variant = 'default' } = $props();
</script>

<button class={{
  btn: true,
  'btn-primary': variant === 'primary',
  'btn-secondary': variant === 'secondary',
  'btn-ghost': variant === 'ghost'
}}>{@render children?.()}</button>
```
