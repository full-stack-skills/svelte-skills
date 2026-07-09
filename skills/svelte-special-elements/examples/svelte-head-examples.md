# svelte:head Examples

`<svelte:head>` lets you insert elements into `document.head`. During server-side rendering, head content is exposed separately from the body content (so frameworks can put it in the HTML `<head>` rather than inlining it into the body).

It may only appear at the top level of your component — never inside a block or element.

```svelte
<svelte:head>
  <title>Page Title</title>
  <meta name="description" content="..." />
</svelte:head>
```

---

## 1. Basic Title + Description

The most common SEO pattern — set the document title and meta description from component state.

```svelte
<script>
  let { post } = $props();
</script>

<svelte:head>
  <title>{post.title} — My Blog</title>
  <meta name="description" content={post.excerpt} />
</svelte:head>

<article>
  <h1>{post.title}</h1>
  <p>{post.excerpt}</p>
</article>
```

---

## 2. Open Graph + Twitter Card Tags

Drive social-share previews from component data.

```svelte
<script>
  let { post } = $props();
</script>

<svelte:head>
  <title>{post.title}</title>

  <!-- Open Graph -->
  <meta property="og:title" content={post.title} />
  <meta property="og:description" content={post.excerpt} />
  <meta property="og:image" content={post.coverImage} />
  <meta property="og:type" content="article" />
  <meta property="og:url" content={post.canonicalUrl} />

  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:title" content={post.title} />
  <meta name="twitter:description" content={post.excerpt} />
  <meta name="twitter:image" content={post.coverImage} />
</svelte:head>
```

---

## 3. Dynamic Stylesheet Loading

Add a per-page stylesheet to the head. Useful for theme-specific or route-specific styles that should not be global.

```svelte
<script>
  let { theme = 'default' } = $props();
</script>

<svelte:head>
  {#if theme === 'dark'}
    <link rel="stylesheet" href="/themes/dark.css" />
  {:else if theme === 'high-contrast'}
    <link rel="stylesheet" href="/themes/high-contrast.css" />
  {/if}
</svelte:head>

<main class="theme-{theme}">
  <slot />
</main>
```

---

## 4. JSON-LD Structured Data

Inject structured data for SEO. Svelte renders arbitrary elements inside `svelte:head`, so you can use `<script type="application/ld+json">`.

```svelte
<script>
  let { product } = $props();
</script>

<svelte:head>
  <title>{product.name} | Shop</title>
  <script type="application/ld+json">
    {JSON.stringify({
      '@context': 'https://schema.org',
      '@type': 'Product',
      name: product.name,
      image: product.image,
      description: product.description,
      offers: {
        '@type': 'Offer',
        price: product.price,
        priceCurrency: 'USD'
      }
    })}
  </script>
</svelte:head>
```

> Note: the inner `{}` braces inside the JSON literal are JS expressions — you may need to escape them or build the string first to satisfy the Svelte parser.

---

## 5. Dynamic `<link rel="canonical">`

Avoid duplicate-content SEO penalties by setting the canonical URL per page.

```svelte
<script>
  import { page } from '$app/stores'; // SvelteKit
  let { post } = $props();
</script>

<svelte:head>
  <link rel="canonical" href={`https://example.com/blog/${post.slug}`} />
  <title>{post.title}</title>
</svelte:head>
```

---

## 6. Inline Critical CSS

Inject per-route critical CSS into the head to avoid FOUC.

```svelte
<svelte:head>
  {@html `<style>
    .landing-hero { background: #111; color: #fff; padding: 4rem 2rem; }
    .landing-hero h1 { font-size: 3rem; }
  </style>`}
</svelte:head>

<section class="landing-hero">
  <h1>Welcome</h1>
</section>
```

---

## 7. Prefetch / Preload Resources

Tell the browser to prefetch or preload assets the next page is going to need.

```svelte
<script>
  let { nextRoute } = $props();
</script>

<svelte:head>
  <link rel="prefetch" href={nextRoute} />
  <link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin />
</svelte:head>
```

---

## 8. Meta Tags from Reactive State

When the underlying state changes, Svelte automatically updates the head to match.

```svelte
<script>
  let count = $state(0);
</script>

<svelte:head>
  <title>Count: {count}</title>
  <meta name="x-count" content={String(count)} />
</svelte:head>

<button onclick={() => count++}>
  Increment ({count})
</button>
```

Open devtools and watch the `<title>` and `<meta name="x-count">` update as you click.

---

## 9. Multiple Head Blocks in One Component

You can include more than one `<svelte:head>` per component — they all merge into `document.head`. Useful for splitting concerns.

```svelte
<svelte:head>
  <title>My Page</title>
</svelte:head>

<svelte:head>
  <link rel="stylesheet" href="/page.css" />
</svelte:head>

<svelte:head>
  <meta name="description" content="..." />
</svelte:head>
```

---

## 10. SSR Head Extraction

In a SvelteKit or custom SSR setup, the `head` content is exposed separately from the `body`. A SvelteKit `+page.svelte` (or any component using `svelte:head`) renders into the document `<head>` automatically. With `render(...)` from `svelte/server`, you receive the `head` and `body` strings:

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

---

## 11. Conditional Head Content

Toggle head content based on a state flag (e.g. show a "noindex" meta when in preview mode).

```svelte
<script>
  let { post, isPreview = false } = $props();
</script>

<svelte:head>
  <title>{post.title}</title>

  {#if isPreview}
    <meta name="robots" content="noindex,nofollow" />
  {/if}
</svelte:head>
```

---

## 12. Resetting to an Empty Head (e.g. on 404)

If you need to clear the previous title before setting a new one, you can render an empty title. Svelte replaces matching elements rather than appending duplicates.

```svelte
<script>
  import { page } from '$app/stores';
</script>

<svelte:head>
  <title>{$page.status === 404 ? 'Page not found' : $page.data.title}</title>
  <meta name="description" content={$page.status === 404 ? 'Not found' : $page.data.description} />
</svelte:head>
```

---

## Gotchas

- **Top-level only.** `<svelte:head>` cannot appear inside `{#if}`, `{#each}`, `{#await}`, or any other element.
- **SSR splits head from body.** Head content is not inlined into the rendered body HTML — your framework must place it in `<head>`.
- **Updates are reactive.** Bind the content to `$state` / `$derived` and the head updates automatically.
- **Multiple `<svelte:head>` blocks per file are allowed.** They all merge.
- **Matching elements are updated, not duplicated.** Two `<title>` elements in the same render will conflict; only the last one wins.
- **`<svelte:head>` cannot contain other `svelte:` special elements** (e.g. no nested `<svelte:window>`).
