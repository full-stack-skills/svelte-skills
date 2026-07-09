# Debug and HTML Examples

Demonstrates `{@html}`, `{@debug}`, and `{@const}` directives in Svelte 5.

## 1. `{@html}` Basic Usage

Render raw HTML (trusted content only):

```svelte
<script>
  let { content = '<p>Hello, <strong>world</strong>!</p>' } = $props();
</script>

<div>
  {@html content}
</div>
```

---

## 2. `{@html}` with Trusted Content

Sanitizing HTML before rendering:

```svelte
<script>
  import DOMPurify from 'dompurify'; // External sanitization

  let { rawHtml = '' } = $props();

  const sanitized = $derived(DOMPurify.sanitize(rawHtml));
</script>

<div>
  {@html sanitized}
</div>
```

> **Security Note**: Always sanitize user-provided HTML before using `{@html}`.

---

## 3. `{@html}` with Conditional Content

Dynamic content rendering:

```svelte
<script>
  let { isMarkdown = false, content = '' } = $props();
  import { marked } from 'marked';

  const rendered = $derived(isMarkdown ? marked.parse(content) : content);
</script>

<article>
  {@html rendered}
</article>
```

---

## 4. `{@debug}` with Variable Inspection

Inspect variables during development:

```svelte
<script>
  let count = $state(0);
  let user = $state({ name: 'Alice', age: 30 });
</script>

{@debug count, user}

<button onclick={() => count++}>
  Count: {count}
</button>
```

When `count` or `user` changes, Svelte logs to console.

---

## 5. `{@debug}` with Custom Label

Using debug with computed values:

```svelte
<script>
  let items = $state([1, 2, 3, 4, 5]);
  const sum = $derived(items.reduce((a, b) => a + b, 0));
</script>

{@debug items, sum}

<ul>
  {#each items as item}
    <li>{item}</li>
  {/each}
</ul>
```

---

## 6. `{@const}` in Each Blocks

Define constants within loops:

```svelte
<script>
  let { products = [] } = $props();
</script>

<table>
  {#each products as product}
    {@const discount = product.price * 0.1}
    {@const finalPrice = product.price - discount}
    <tr>
      <td>{product.name}</td>
      <td>${product.price.toFixed(2)}</td>
      <td>${finalPrice.toFixed(2)}</td>
    </tr>
  {/each}
</table>
```

---

## 7. `{@const}` with Computed Values

Complex calculations in templates:

```svelte
<script>
  let { width = 100, height = 100 } = $props();
</script>

{@const area = width * height}
{@const perimeter = 2 * (width + height)}
{@const isSquare = width === height}

<div class="rectangle">
  <p>Area: {area}</p>
  <p>Perimeter: {perimeter}</p>
  <p>Square: {isSquare ? 'Yes' : 'No'}</p>
</div>
```

---

## 8. `{@const}` with Snippet Parameters

Constants in snippet blocks:

```svelte
<script>
  let { items = [] } = $props();
</script>

{#snippet renderItem(item)}
  {@const index = items.indexOf(item)}
  {@const isFirst = index === 0}
  {@const isLast = index === items.length - 1}
  <div class:is-first={isFirst} class:is-last={isLast}>
    <slot />
  </div>
{/snippet}
```

---

## 9. Combined Debug Example

```svelte
<script>
  let count = $state(0);
  let user = $state({ name: 'Bob', active: true });

  function reset() {
    count = 0;
    user = { ...user, active: false };
  }
</script>

{@debug count, user.name, user.active}

<button onclick={reset}>Reset</button>
<button onclick={() => count++}>Increment</button>
```

---

## 10. `{@html}` in Lists

Rendering HTML content in list items:

```svelte
<script>
  let { posts = [] } = $props();
</script>

<div class="posts">
  {#each posts as post}
    <article>
      <h2>{post.title}</h2>
      {@html post.excerpt}
    </article>
  {/each}
</div>
```
