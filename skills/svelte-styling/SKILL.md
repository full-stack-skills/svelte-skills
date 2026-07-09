---
name: svelte-styling
description: Svelte 5 样式技能。当用户需要使用作用域样式、:global、CSS 自定义属性、class 指令、嵌套 style、或理解 Svelte 的样式隔离机制时使用。
---

# Svelte Styling Reference (Svelte 5)

本技能覆盖 Svelte 组件的样式系统，包括作用域样式、全局样式、CSS 自定义属性和 class 处理。

## When to use this skill

当用户需要编写 Svelte 组件的样式、处理样式隔离、CSS 变量传递、或理解 Svelte 编译器如何处理样式时使用本技能。

## Critical: Scoped Styles

`<style>` 块中的 CSS 默认**作用域化**——仅影响当前组件内的元素：

```svelte
<style>
  p { color: burlywood; }
  .container { padding: 1em; }
</style>
```

编译时 Svelte 为作用域内的选择器自动添加哈希类（如 `svelte-123xyz`）。

### 特异性

Scoped 选择器额外增加 **0-1-0** 特异性（来自作用域类），可能覆盖全局相同选择器：

```css
/* 全局 */
p { color: blue; }

/* 组件内（scoped，优先级更高） */
p { color: red; }
```

### Scoped Keyframes

`@keyframes` 名称也会自动作用域化：

```svelte
<style>
  @keyframes bounce {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-20px); }
  }
  .animated { animation: bounce 1s; }
</style>
```

## Critical: Global Styles

### :global() 单选择器

```svelte
<style>
  :global(body) { margin: 0; }          /* 全局 body */
  div :global(strong) { color: goldenrod; } /* 嵌套全局 */
  p:global(.big.red) { font-weight: bold; } /* 带条件 */
</style>
```

### :global {...} 块（多选择器）

```svelte
<style>
  :global {
    p { margin: 1em; }
    .theme-dark { --bg: #111; }
    div :global { .foo .bar { color: red; } }
  }
</style>
```

### 全局 Keyframes

在 keyframe 名称前加 `-global-`：

```svelte
<style>
  @keyframes -global-my-animation-name {
    0% { opacity: 0; }
    100% { opacity: 1; }
  }
</style>
```

编译后 `-global-` 前缀被移除，成为 `my-animation-name` 可在其他组件中引用。

## Critical: CSS Custom Properties

### 传入自定义属性

```svelte
<Slider bind:value --track-color="black" --thumb-color="rgb(255 0 0)" />
```

### 组件内读取

```svelte
<style>
  .track { background: var(--track-color, #aaa); }
  .thumb { background: var(--thumb-color, blue); }
</style>
```

### CSS 自定义属性可级联

无需直接传值，只要父元素定义了 CSS 变量，子组件即可读取：

```css
/* 全局 */
:root { --primary: #ff3e00; }
```

## Critical: class Attribute

Svelte 5.16+ 的 `class` 属性支持对象和数组：

### 对象形式（clsx 风格）

```svelte
<div class={{ cool: isCool, large: isLarge }}>...</div>
<!-- truthy key 加入 class -->
```

### 数组形式

```svelte
<div class={[faded && 'saturate-0 opacity-50', large && 'scale-200']}>...</div>
```

### 混合形式

```svelte
<Button class={['btn', props.class]} />
```

### 类型安全

```svelte
<script lang="ts">
  import type { ClassValue } from 'svelte/elements';
  let { class: cls }: { class: ClassValue } = $props();
</script>
<div class={['original', cls]}>...</div>
```

## Critical: class: Directive（不推荐）

Svelte 5.16 后推荐用 `class={}` 对象/数组形式替代：

```svelte
<!-- 等价 -->
<div class={{ cool, lame: !cool }}>...</div>
<div class:cool class:lame={!cool}>...</div>
```

## Critical: Nested `<style>` Elements

一个组件只能有**一个顶层** `<style>` 块，但可以嵌套 `<style>` 标签（不经过作用域处理，直接插入 DOM）：

```svelte
<div>
  <style>
    /* 原生 style 标签，无作用域 */
    div { color: red; }
  </style>
</div>
```

## Quick Fixes

| 问题 | 解决方案 |
|------|----------|
| 样式不生效 | 检查 scoped 哈希是否覆盖；用 `:global` |
| 全局样式污染 | 将全局样式移到独立 CSS 文件或 `:global` 中 |
| 组件样式传不进去 | 用 CSS 自定义属性（`--var`）而非 scoped 选择器 |
| Tailwind 不生效 | 用 `class={['tailwind-class', props.class]}` 组合 |
| 动画 keyframe 冲突 | 用 `-global-` 前缀使 keyframe 全局唯一 |

## Gotchas

1. **Scoped 样式隔离** — 通过哈希类实现，不会真正"隐藏"样式（查看页面源码可见）
2. **`{@html}` 内容不受 scoped 影响** — 插入的 HTML 无作用域类
3. **CSS 变量不需要 scoped** — `var(--custom)` 天然跨作用域
4. **组件根元素直接接收 class prop** — `<div class:myclass>` 会添加到根元素

## FAQ

**Q: 如何让样式影响子组件内部？**
A: 两种方式：① 父组件传入 CSS 自定义属性（推荐）；② 用 `:global`（影响全局）。

**Q: Svelte 编译后 class 名称是什么？**
A: 形如 `svelte-123xyz` 的哈希，组件级唯一。

**Q: 可以同时用 scoped 和全局样式吗？**
A: 可以，同一 `<style>` 中可混合使用 scoped 选择器和 `:global`。

## Examples

| File | Description |
|------|-------------|
| [examples/README.md](./examples/README.md) | Entry point listing all example files with descriptions |
| [examples/scoped-styles.md](./examples/scoped-styles.md) | Scoped styles, :global(), CSS custom properties, class directive patterns |
| [examples/tailwind.md](./examples/tailwind.md) | Tailwind CSS integration with dynamic classes and dark mode |

## References

| File | Description |
|------|-------------|
| [references/README.md](./references/README.md) | Entry point listing all reference files with descriptions |
| [references/scoped-deep.md](./references/scoped-deep.md) | Deep dive into scoping mechanism, :global() variants, style isolation |
| [references/css-custom-properties.md](./references/css-custom-properties.md) | CSS Custom Properties: passing, reading, cascading, dynamic updates |
| [references/class-directive.md](./references/class-directive.md) | class vs class: directive, ClassValue types, clsx patterns
