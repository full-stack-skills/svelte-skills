# Images

Three approaches: Vite's built-in handling, `@sveltejs/enhanced-img`, and dynamic CDN loading.

## 1. Vite's built-in (asset import)

```svelte
<script>
  import logo from '$lib/assets/logo.png';
</script>
<img alt="Logo" src={logo} />
```

Vite hashes the filename (cache-busting) and inlines small assets as base64.

## 2. `@sveltejs/enhanced-img` — install

```sh
npm i -D @sveltejs/enhanced-img
```

```js
// vite.config.js — order matters
import { enhancedImages } from '@sveltejs/enhanced-img';
import { sveltekit } from '@sveltejs/kit/vite';
export default {
  plugins: [enhancedImages(), sveltekit()]
};
```

## 3. Basic `<enhanced:img>`

```svelte
<enhanced:img src="./photo.jpg" alt="Hero" />
```

Builds a `<picture>` with `avif` + `webp` sources, multiple sizes, intrinsic dimensions.

## 4. Specify `sizes` for responsive images

```svelte
<enhanced:img src="./hero.jpg" alt="Hero" sizes="min(1280px, 100vw)" />
```

Generates images at 540, 640, 750, 828, 1080, 1200, 1920, 2048, 3840 widths.

## 5. Custom widths

```svelte
<enhanced:img
  src="./hero.jpg?w=1280;640;400"
  sizes="(min-width:1920px) 1280px, (min-width:1080px) 640px, (min-width:768px) 400px"
  alt="Hero"
/>
```

## 6. Dynamic image choice

```svelte
<script>
  import MyImage from './photo.jpg?enhanced';
</script>
<enhanced:img src={MyImage} alt="..." />
```

The `?enhanced` query tells Vite to apply imagetools.

## 7. Iterate images with `import.meta.glob`

```svelte
<script>
  const images = import.meta.glob(
    '/src/lib/gallery/*.{jpg,JPG,png,PNG,webp,WEBP}',
    { eager: true, query: { enhanced: true } }
  );
</script>

{#each Object.entries(images) as [path, mod]}
  <enhanced:img src={mod.default} alt={path.split('/').pop()} />
{/each}
```

## 8. Per-image transforms

```svelte
<enhanced:img src="./photo.jpg?blur=15" alt="..." />
<enhanced:img src="./photo.jpg?quality=60" alt="..." />
<enhanced:img src="./photo.jpg?rotate=90" alt="..." />
```

See the [imagetools directives](https://github.com/JonasKruckenberg/imagetools/blob/main/docs/directives.md) for the full list.

## 9. Dynamic CDN loading — `@unpic/svelte`

For images not available at build time (CMS, DB):

```sh
npm i @unpic/svelte
```

```svelte
<script>
  import { Image } from '@unpic/svelte';
</script>

<Image
  src="https://cdn.example.com/photo.jpg"
  width={800}
  height={600}
  alt="Photo"
  layout="constrained"
/>
```

CDN picks the best format based on User-Agent. Works without build-time data.

## 10. Vercel image optimization

If deploying to Vercel, configure the adapter:

```js
// svelte.config.js
adapter({
  images: {
    sizes: [640, 828, 1200, 1920, 3840],
    formats: ['image/avif', 'image/webp'],
    minimumCacheTTL: 300,
    domains: ['my-cms.example.com']
  }
})
```

Use `<img>` with `/\_vercel/image?url=...&w=...` URLs for dynamic optimization.

## 11. LCP image — prioritize

```svelte
<enhanced:img
  src="./hero.jpg"
  alt="..."
  fetchpriority="high"
/>
```

Don't add `loading="lazy"` to LCP images.

## 12. Avoid layout shift

`@sveltejs/enhanced-img` auto-adds `width` and `height`. For Vite's built-in or CDN-loaded images, set them manually:

```svelte
<img src={logo} alt="Logo" width="200" height="50" />
```

Or use CSS aspect-ratio containers.

## 13. CSS selector caveat

If you style `<enhanced:img>` by tag, escape the colon:

```svelte
<style>
  enhanced\:img { display: block; }
</style>
```

## See also

- [Images reference](../references/images-reference.md)
