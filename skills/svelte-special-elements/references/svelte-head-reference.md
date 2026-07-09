# svelte:head Reference

`<svelte:head>` lets you insert elements into `document.head`. During server-side rendering, head content is exposed **separately** from the main body content (so frameworks can place it in the HTML `<head>` rather than inlining it into the body).

```svelte
<svelte:head>
  <title>Hello world!</title>
  <meta name="description" content="This is where the description goes for SEO" />
</svelte:head>
```

---

## Syntax

```svelte
<svelte:head>
  <!-- any HTML / SVG elements -->
  <title>...</title>
  <meta ... />
  <link ... />
  <style>...</style>
  <script>...</script>
</svelte:head>
```

## Constraints

- **Top-level only.** `<svelte:head>` must appear at the top level of a component — never inside a block (`{#if}`, `{#each}`, `{#await}`) or another element. The same constraint applies to `<svelte:window>`, `<svelte:document>`, and `<svelte:body>`.
- **Cannot contain other `svelte:` special elements.** No nested `<svelte:window>`, etc.
- **Multiple blocks are allowed.** You can have several `<svelte:head>` blocks in one component; they all merge into `document.head`.

## Common Elements

| Element | Use |
|---------|-----|
| `<title>` | Document title |
| `<meta name="..." content="..." />` | SEO, social, robots directives |
| `<meta property="og:..." content="..." />` | Open Graph metadata |
| `<link rel="stylesheet" href="..." />` | Per-page stylesheets |
| `<link rel="canonical" href="..." />` | Canonical URL |
| `<link rel="prefetch\|preload" href="..." />` | Resource hints |
| `<style>` | Inline critical CSS |
| `<script type="application/ld+json">` | JSON-LD structured data |
| `<base href="..." />` | Base URL for relative links |

## Reactivity

`<svelte:head>` content is fully reactive. Bind it to `$state` / `$derived` and Svelte will update the head automatically as the values change. When the same kind of element exists in two places (e.g. two `<title>` tags), Svelte replaces the previous one rather than appending a duplicate.

```svelte
<script>
  let count = $state(0);
</script>

<svelte:head>
  <title>Count: {count}</title>
</svelte:head>

<button onclick={() => count++}>Increment ({count})</button>
```

## SSR Behavior

In server-side rendering, head content is **exposed separately** from the body content. The exact shape of the API depends on the framework:

- **`svelte/server` (low-level):** `render(...)` returns `{ head, body }`.
- **SvelteKit:** the `head` content is automatically hoisted into the document `<head>` via the framework's `app.html`.

```js
// server.js
import { render } from 'svelte/server';
import App from './App.svelte';

const { head, body } = await render(App);

const html = `
<!doctype html>
<html>
  <head>${head}</head>
  <body>${body}</body>
</html>
`;
```

> During SSR, the head content is **not** inlined into the body HTML. If you see head content showing up in the body in your SSR output, your framework is misconfigured.

## Hydration

On hydration, Svelte matches the existing DOM elements in `document.head` to the rendered output and adopts them. You can include `<svelte:head>` content in your server-rendered HTML and Svelte will reuse it without re-creating it on the client.

## Common Pitfalls

1. **Don't put it inside a block.** `<svelte:head>` must be top-level. If you need conditional head content, use `{#if}` *inside* `<svelte:head>`:

   ```svelte
   <svelte:head>
     {#if isPreview}
       <meta name="robots" content="noindex" />
     {/if}
   </svelte:head>
   ```

2. **Don't render two `<title>`s in the same render.** Svelte will only keep the last one.

3. **JSON-LD requires `JSON.stringify`.** The contents of `<script type="application/ld+json">` are parsed as Svelte, not as raw HTML — escape braces or use `{@html ...}` if needed.

4. **Head content is reactive, but you cannot bind events to it** (no `onclick` on a `<title>`, etc.). Use `$effect` if you need imperative side-effects on head changes.

5. **Hydration requires SSR.** If you ship a `<svelte:head>` block in a CSR-only app, it will work, but you'll see the title flash from empty to populated on mount. Render the head content into your shell HTML for a flicker-free experience.

## When to Use

| Use case | Element to use |
|----------|----------------|
| Set the page `<title>` from a route param | `<title>{post.title}</title>` |
| SEO meta description | `<meta name="description" />` |
| Open Graph / Twitter cards | `<meta property="og:..." />` |
| Per-page theme stylesheet | `<link rel="stylesheet" />` |
| Canonical URL | `<link rel="canonical" />` |
| Resource hints | `<link rel="prefetch" \| rel="preload" />` |
| Structured data | `<script type="application/ld+json">` |
| Inline critical CSS | `<style>` |
