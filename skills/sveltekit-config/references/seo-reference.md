# SEO reference

SvelteKit provides SSR, normalized URLs, and good defaults. Manual setup for full SEO control.

## What SvelteKit handles out of the box

### SSR

Pages are server-rendered by default. Crawlers can index fully-rendered content. Don't disable SSR without a good reason.

### Normalized trailing slashes

SvelteKit redirects `/about/` to `/about` (or vice versa, depending on `trailingSlash`). Duplicate URLs hurt SEO; normalization fixes this.

### Asset preloading

`modulepreload` links reduce load time → better Core Web Vitals → better rankings.

### Code splitting

Only ships code for the current route → faster TTI → better CWV.

## What to do manually

### 1. Per-page meta tags

```svelte
<svelte:head>
  <title>{data.title} — Site Name</title>
  <meta name="description" content={data.excerpt} />
  <link rel="canonical" href="https://example.com{page.url.pathname}" />
</svelte:head>
```

Titles: 50-60 chars. Descriptions: 150-160 chars.

### 2. Open Graph (Facebook, LinkedIn, Slack)

```svelte
<svelte:head>
  <meta property="og:type" content="article" />
  <meta property="og:title" content={data.title} />
  <meta property="og:description" content={data.excerpt} />
  <meta property="og:url" content="https://example.com{page.url.pathname}" />
  <meta property="og:image" content={data.coverUrl} />
  <meta property="og:site_name" content="My Site" />
  <meta property="og:locale" content="en_US" />
</svelte:head>
```

Image: 1200x630 minimum.

### 3. Twitter Card

```svelte
<svelte:head>
  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:site" content="@yourhandle" />
  <meta name="twitter:creator" content="@author" />
  <meta name="twitter:title" content={data.title} />
  <meta name="twitter:description" content={data.excerpt} />
  <meta name="twitter:image" content={data.coverUrl} />
</svelte:head>
```

### 4. JSON-LD structured data

Critical for rich results (articles, products, recipes, FAQs).

```svelte
{@html `<script type="application/ld+json">${JSON.stringify({
  '@context': 'https://schema.org',
  '@type': 'Article',
  headline: data.title,
  image: data.coverUrl,
  datePublished: data.publishedAt,
  author: { '@type': 'Person', name: data.author },
  publisher: { '@type': 'Organization', name: 'My Site', logo: { '@type': 'ImageObject', url: 'https://example.com/logo.png' } }
})}</script>`}
```

Test at https://search.google.com/test/rich-results

### 5. Sitemap

```js
// src/routes/sitemap.xml/+server.js
export async function GET({ url }) {
  const pages = [
    { loc: '/', priority: 1.0, changefreq: 'weekly' },
    { loc: '/about', priority: 0.8 },
    { loc: '/blog', priority: 0.9, changefreq: 'daily' }
    // dynamic pages
  ];
  
  const xml = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
${pages.map(p => `  <url>
    <loc>${url.origin}${p.loc}</loc>
    <changefreq>${p.changefreq || 'monthly'}</changefreq>
    <priority>${p.priority || 0.5}</priority>
  </url>`).join('\n')}
</urlset>`;
  
  return new Response(xml, { headers: { 'Content-Type': 'application/xml' } });
}
```

Submit to Google Search Console and Bing Webmaster Tools.

### 6. robots.txt

```js
// src/routes/robots.txt/+server.js
export function GET({ url }) {
  return new Response(`User-agent: *
Allow: /
Disallow: /admin/
Disallow: /api/
Sitemap: ${url.origin}/sitemap.xml
`);
}
```

### 7. Canonical URLs

Prevent duplicate content penalties:

```svelte
<svelte:head>
  <link rel="canonical" href="https://example.com{page.url.pathname}" />
</svelte:head>
```

Even when accessed via www/non-www or with query params.

## Common patterns

### SEO data from `load`

```js
// src/routes/+layout.server.js
export const load = ({ url }) => ({
  seo: {
    title: 'My App',
    description: 'Default description',
    url: url.href
  }
});
```

```svelte
<!-- src/routes/+layout.svelte -->
<svelte:head>
  <title>{data.seo.title}</title>
  <meta name="description" content={data.seo.description} />
</svelte:head>
```

Override per page:

```js
// src/routes/about/+page.server.js
export const load = async ({ parent }) => {
  const { seo } = await parent();
  return {
    seo: { ...seo, title: 'About — My App' }
  };
};
```

### hreflang for multi-language

```svelte
<svelte:head>
  <link rel="alternate" hreflang="en" href="https://example.com/en/about" />
  <link rel="alternate" hreflang="de" href="https://example.com/de/about" />
  <link rel="alternate" hreflang="x-default" href="https://example.com/about" />
</svelte:head>
```

### AMP (rare)

```js
// svelte.config.js
export default {
  kit: { inlineStyleThreshold: Infinity }
};
```

```js
// src/routes/+layout.server.js
export const csr = false;
```

```html
<!-- src/app.html -->
<html ⚡>
```

```js
// src/hooks.server.js
import * as amp from '@sveltejs/amp';
export async function handle({ event, resolve }) {
  let buffer = '';
  return resolve(event, {
    transformPageChunk: ({ html, done }) => {
      buffer += html;
      if (done) return amp.transform(buffer);
    }
  });
}
```

## Performance and SEO

Core Web Vitals directly affect rankings:
- **LCP** (Largest Contentful Paint) — < 2.5s
- **FID / INP** (Interaction to Next Paint) — < 200ms
- **CLS** (Cumulative Layout Shift) — < 0.1

Optimize:
- Preload LCP image
- Minimize JS / lazy-load
- Reserve space for images (width/height)

## Verification

- [Google Search Console](https://search.google.com/search-console)
- [Bing Webmaster Tools](https://www.bing.com/webmasters)
- [PageSpeed Insights](https://pagespeed.web.dev/)
- [Rich Results Test](https://search.google.com/test/rich-results)
- [Lighthouse SEO audit](https://web.dev/lighthouse-seo/)

## See also

- [examples/seo.md](../examples/seo.md)
- [examples/performance.md](../examples/performance.md)
