# {@render} / {@html} / {@debug} / {@const} / {@attach}

Svelte 5 中所有以 `@` 开头的特殊模板标签。

## 1. {@render} 调用 Snippet

```svelte
{#snippet sum(a, b)}
  <p>{a} + {b} = {a + b}</p>
{/snippet}

{@render sum(1, 2)}
{@render sum(3, 4)}
{@render sum(5, 6)}
```

> 表达式可以是标识符或任意 JS 表达式：`{@render (cond ? a : b)()}`。

## 2. {@render} + children

```svelte
<!-- Modal.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte';

  let {
    title,
    children
  }: { title: string; children: Snippet } = $props();
</script>

<div class="modal">
  <h2>{title}</h2>
  {@render children()}
</div>

<!-- Parent.svelte -->
<Modal title="Info">
  {#snippet children()}
    <p>content</p>
  {/snippet}
</Modal>
```

## 3. {@render} 条件渲染

```svelte
<script>
  let show = $state(true);
</script>

{#snippet panel(content)}
  <div class="panel">{content}</div>
{/snippet}

{#if show}
  {@render panel('Content')}
{/if}
```

## 4. {@render} 可选 snippet

```svelte
<!-- children 是可选 prop -->
{@render children?.()}

<!-- 或用 #if 提供 fallback -->
{#if children}
  {@render children()}
{:else}
  <p>fallback content</p>
{/if}
```

## 5. {@html} 原始 HTML

```svelte
<script>
  let html = '<strong>Bold</strong> and <em>italic</em>';
</script>

{@html html}
```

⚠️ **安全警告**：只对可信内容使用，XSS 风险：
```svelte
<!-- ❌ 危险：用户输入未经转义 -->
{@html userInput}

<!-- ✅ 安全 -->
{@html sanitize(userInput)}
```

注意：
- 表达式必须**是完整、独立**的 HTML 字符串 — 跨 `{@html}` 的标签不会拼接
- 不会编译 Svelte 代码
- **不受 Svelte scoped 样式影响** — 需用 `:global` 选择器

```svelte
<!-- ❌ scoped 样式对 {@html} 内的元素不生效 -->
<style>
  article a { color: hotpink }  /* 不会匹配 {@html} 内的 a */
</style>

<!-- ✅ 用 :global -->
<style>
  article :global(a) { color: hotpink }
</style>
```

## 6. {@debug} 开发调试

```svelte
<script>
  let count = $state(0);
  let user = $state({ name: 'Ada' });
</script>

{@debug count, user}
<h1>{user.name}</h1>
```

`{@debug var1, var2, ...}` 会在任一变量变化时打印当前值，并在 devtools 打开时暂停。

无参数形式：
```svelte
{@debug}
<!-- 任意响应式 state 变化时触发 -->
```

**限制**：只接受变量名（逗号分隔），不接受表达式。

```svelte
<!-- ✅ -->
{@debug user}
{@debug user1, user2}

<!-- ❌ 不编译 -->
{@debug user.firstname}
{@debug myArray[0]}
{@debug !isReady}
{@debug typeof user === 'object'}
```

## 7. {@const} 模板常量

> ⚠️ Legacy 语法。Svelte 5.56+ 推荐用 `{const x = ...}` 声明标签。

```svelte
{#each boxes as box}
  {@const area = box.width * box.height}
  {box.width} * {box.height} = {area}
{/each}
```

**仅允许**作为以下块的直接子元素：
- `{#if ...}`
- `{#each ...}`
- `{#snippet ...}`
- `<Component />`
- `<svelte:boundary>`

## 8. {let/const} 声明标签（Svelte 5.56+）

```svelte
{#each boxes as box}
  {const area = box.width * box.height}
  {const label = `${box.width} ⨉ ${box.height} = ${area}`}
  <p>{label}</p>
{/each}
```

可在组件**任意位置**使用。引用外部值，作用域为词法兄弟/子节点。

### 响应式声明

```svelte
<script>
  let user = $state({ name: 'Svelte' });
  let editing = $state(false);
</script>

{#if editing}
  {let name = $state(user.name)}
  {const greeting = $derived(`Hello ${name}`)}

  <input bind:value={name} />
  <p>{greeting}</p>
{/if}
```

## 9. {@attach}（替代 use:action）

```svelte
<script>
  import { initializeChart } from './chart.js';
  let data = $state([1, 2, 3]);
</script>

<div {@attach initializeChart(data)}></div>
```

详细见 `attachments.md`。

## 10. 组合示例

```svelte
{#snippet itemCard(item)}
  {@const tag = item.featured ? 'Featured' : 'Standard'}
  <div class="card">
    <span class="tag">{tag}</span>
    <h3>{item.title}</h3>
    {@html item.body}  <!-- 假定已 sanitize -->
    <pre>{@debug}</pre>
  </div>
{/snippet}

{#each items as item (item.id)}
  {@render itemCard(item)}
{/each}
```
