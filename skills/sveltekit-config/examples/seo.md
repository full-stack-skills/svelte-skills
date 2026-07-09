# SEO

SvelteKit's defaults (SSR, normalized trailing slashes) cover the basics. Manual setup for full control.

## 1. Unique title and description

```svelte
<!-- src/routes/blog/[slug]/+page.svelte -->
<script>
  import { page } from '$app/state';
  let { data } = $props();
</script>

<svelte:head>
  <title>{data.post.title} — My Blog</title>
  <meta name="description" content={data.post.excerpt} />
</svelte:head>
```

## 2. Open Graph + Twitter

```svelte
<svelte:head>
  <title>{data.post.title}</title>
  <meta name="description" content={data.post.excerpt} />

  <!-- Open Graph -->
  <meta property="og:type" content="article" />
  <meta property="og:title" content={data.post.title} />
  <meta property="og:description" content={data.post.excerpt} />
  <meta property="og:image" content={data.post.coverUrl} />
  <meta property="og:url" content={page.url.href} />
  <meta property="og:site_name" content="My Blog" />

  <!-- Twitter -->
  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:title" content={data.post.title} />
  <meta name="twitter:description" content={data.post.excerpt} />
  <meta name="twitter:image" content={data.post.coverUrl} />

  <!-- Canonical -->
  <link rel="canonical" href={page.url.href} />
</svelte:head>
```

## 3. JSON-LD structured data

```svelte
<svelte:head>
  {@html `<script type="application/ld+json">${JSON.stringify({
    '@context': 'https://schema.org',
    '@type': 'Article',
    headline: data.post.title,
    image: data.post.coverUrl,
    datePublished: data.post.publishedAt,
    author: { '@type': 'Person', name: data.post.author }
  })}</script>`}
</svelte:head>
```

Or import a JSON-LD component:

```svelte
<JsonLd data={{
  '@context': 'https://schema.org',
  '@type': 'Article',
  headline: data.post.title
}} />
```

## 4. SEO data from `load`

Common pattern: load SEO data in `+page.server.js`, render it in root layout's `<svelte:head>`.

```js
// src/routes/+layout.server.js
export const load = async ({ url }) => ({
  seo: {
    title: 'My App',
    description: 'Default description',
    url: url.href
  }
});
```

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  let { data, children } = $props();
</script>

<svelte:head>
  <title>{data.seo.title}</title>
  <meta name="description" content={data.seo.description} />
</svelte:head>

{@render children()}
```

Pages can override:

```js
// src/routes/about/+page.server.js
export const load = ({ parent }) => parent().then(p => ({
  seo: { ...p.seo, title: 'About — My App', description: 'About us' }
}));
```

## 5. Sitemap endpoint

```js
// src/routes/sitemap.xml/+server.js
export async function GET({ url }) {
  const pages = ['', 'about', 'blog', 'contact'];
  const base = url.origin;
  
  const xml = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
${pages.map(p => `  <url>
    <loc>${base}/${p}</loc>
    <changefreq>weekly</changefreq>
  </url>`).join('\n')}
</urlset>`;

  return new Response(xml, {
    headers: { 'Content-Type': 'application/xml' }
  });
}
```

For dynamic content, fetch all entries and emit `<url>` for each.

## 6. `robots.txt`

```js
// src/routes/robots.txt/+server.js
export function GET({ url }) {
  return new Response(`User-agent: *
Allow: /
Sitemap: ${url.origin}/sitemap.xml
`);
}
```

## 7. Canonical URLs

Always set `<link rel="canonical">` to avoid duplicate-content penalties:

```svelte
<svelte:head>
  <link rel="canonical" href={`https://example.com${page.url.pathname}`} />
</svelte:head>
```

## 8. Prerender + SEO win

Pages with `export const prerender = true` are served instantly from CDN — better TTFB, better CWV, better rankings.

## See also

- [SEO reference](../references/seo-reference.md)
