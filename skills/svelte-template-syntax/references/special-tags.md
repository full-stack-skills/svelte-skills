# Special Template Tags

## {@html}

渲染 HTML 字符串（不做转义）：

```svelte
<script>
  let html = '<strong>Bold</strong>';
</script>

{@html html}
```

⚠️ **安全警告**：只用于可信内容：
```svelte
<!-- ❌ XSS 风险 -->
{@html userProvidedContent}

<!-- ✅ 确保 sanitize 后使用 -->
{@html sanitize(dominatedContent)}
```

## {@debug}

在控制台打印变量（开发时）：

```svelte
<script>
  let count = $state(0);
  let name = $state('Alice');
</script>

<!-- 任意变量 -->
{@debug count, name}

<!-- 等价于（打印所有响应式变量） -->
{@debug}
```

触发条件：`count` 或 `name` 变化时打印。

## {@const}

模板内计算局部常量：

```svelte
{#each items as item}
  {@const total = item.price * item.qty}
  <p>{item.name}: ${total}</p>
{/each}
```

仅在当前 `{#each}` 迭代内有效。

## {@attach}

替代 Svelte 4 `use:action` 的新方式（Svelte 5+）：

```svelte
<script>
  import { myAction } from './actions.js';
</script>

<!-- 替代 use:myAction -->
<div {@attach myAction(data)}></div>
```

## svelte:element（动态组件）

```svelte
<script>
  import Heading from './Heading.svelte';

  let tag = $state('h1');
  let component = $state(Heading);
</script>

<!-- 动态 HTML 标签 -->
<svelte:element this={tag} class="title">Hello</svelte:element>

<!-- 动态组件 -->
<svelte:component this={component} />
```

## svelte:window

绑定 window 事件和属性：

```svelte
<svelte:window
  onkeydown={handleKey}
  onresize={handleResize}
  bind:innerWidth={width}
  bind:innerHeight={height}
/>
```

## svelte:document

绑定 document 事件：

```svelte
<svelte:document
  onvisibilitychange={handleVisibility}
  onselectionchange={handleSelection}
/>
```

## svelte:body

绑定 body 事件：

```svelte
<svelte:body
  onmouseenter={handleMouseEnter}
  onmouseleave={handleMouseLeave}
/>
```
