# Styling Reference

## Scoped Styles 详解

```svelte
<style>
  /* 选择器自动加哈希类 */
  p { color: red; }

  /* 逗号分隔的选择器各自加哈希 */
  h1, h2 { font-weight: bold; }

  /* 伪类/伪元素也加哈希 */
  p:hover { color: blue; }
  p::before { content: '→ '; }

  /* combinators (后代/子/相邻) 各自加哈希 */
  div p { margin: 1em; }
  div > p { padding: 0.5em; }
</style>
```

## :global() 详解

```svelte
<style>
  /* 单选择器全局 */
  :global(body) { margin: 0; }
  :global(*), :global(*::before), :global(*::after) { box-sizing: border-box; }

  /* 嵌套全局 */
  div :global(strong) { color: orange; }
  /* → 匹配 div 内任意深度的 <strong> */

  /* 带选择器的全局 */
  p:global(.important) { font-weight: bold; }

  /* 全局块 */
  :global {
    .theme-dark { --bg: #111; }
    .theme-light { --bg: #fff; }
  }
</style>
```

## CSS Custom Properties 完整用法

```svelte
<!-- 父组件传入 -->
<Card
  --card-bg="#f5f5f5"
  --card-radius="8px"
  --card-padding="16px"
  title="Hello"
/>

<!-- 子组件读取 -->
<style>
  .card {
    background: var(--card-bg, #fff);     /* 默认值 */
    border-radius: var(--card-radius, 4px);
    padding: var(--card-padding, 8px);
  }
</style>
```

## class 属性（clsx 风格）

```svelte
<script>
  let isActive = $state(true);
  let size = $state('large');
</script>

<!-- 对象形式 -->
<div class={{ active: isActive, [`size-${size}`]: true }}>
  Content
</div>

<!-- 数组形式 -->
<div class={['base', isActive && 'active', `size-${size}`]}>
  Content
</div>

<!-- 混合 -->
<div class={['btn', props.class]}>
  <slot />
</div>
```

## Tailwind 集成

```svelte
<script>
  let isPrimary = $state(true);
</script>

<!-- 组合类名 -->
<button class={[
  'px-4 py-2 rounded font-medium transition-colors',
  isPrimary ? 'bg-blue-600 text-white' : 'bg-gray-200 text-gray-800'
]}>
  Button
</button>

<!-- 传入子组件 -->
<BaseButton class="bg-red-500 hover:bg-red-600" />
```
