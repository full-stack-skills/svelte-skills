# Images reference

Three approaches for image handling in SvelteKit: Vite built-in, `@sveltejs/enhanced-img`, and dynamic CDN loading.

## Vite's built-in handling

[Vite automatically processes imported assets](https://vitejs.dev/guide/assets.html):

```svelte
<script>
  import logo from '$lib/assets/logo.png';
</script>

<img alt="Logo" src={logo} />
```

What Vite does:
- Adds content hashes to filenames (cache-busting)
- Inlines assets below `assetsInlineLimit` (~4kb default) as base64
- Works for images, video, audio, fonts

Configuration:

```js
// vite.config.js
export default {
  build: {
    assetsInlineLimit: 8192   // inline if < 8kb
  }
};
```

CSS `url()` references are also processed.

## @sveltejs/enhanced-img

Build-time image optimization:

- Generates modern formats (AVIF, WebP)
- Sets intrinsic `width`/`height` to prevent CLS
- Creates multiple sizes for different viewports
- Strips EXIF data for privacy

### Installation

```sh
npm i -D @sveltejs/enhanced-img
```

```js
// vite.config.js — ORDER MATTERS
import { enhancedImages } from '@sveltejs/enhanced-img';
import { sveltekit } from '@sveltejs/kit/vite';

export default {
  plugins: [
    enhancedImages(),   // must come BEFORE sveltekit()
    sveltekit()
  ]
};
```

First build is slow (image processing); subsequent builds are cached in `./node_modules/.cache/imagetools`.

### Basic usage

```svelte
<enhanced:img src="./photo.jpg" alt="..." />
```

Becomes:

```html
<picture>
  <source type="image/avif" srcset="..." />
  <source type="image/webp" srcset="..." />
  <img src="..." width="..." height="..." alt="..." />
</picture>
```

Provide the highest resolution. Smaller versions are generated automatically. Provide 2x source for HiDPI displays.

### Responsive sizes

```svelte
<enhanced:img src="./hero.jpg" sizes="min(1280px, 100vw)" alt="..." />
```

Generates images at widths: 540, 640, 750, 828, 1080, 1200, 1920, 2048, 3840.

### Custom widths

```svelte
<enhanced:img
  src="./photo.jpg?w=1280;640;400"
  sizes="(min-width:1920px) 1280px, (min-width:1080px) 640px, (min-width:768px) 400px"
  alt="..."
/>
```

### Dynamic images

```svelte
<script>
  import PhotoA from './a.jpg?enhanced';
  import PhotoB from './b.jpg?enhanced';
  let active = $state(PhotoA);
</script>

<enhanced:img src={active} alt="..." />
```

The `?enhanced` query tells Vite to apply imagetools.

### Iterate over many

```svelte
<script>
  const modules = import.meta.glob(
    '/src/lib/gallery/*.{jpg,JPG,png,PNG,webp,WEBP}',
    { eager: true, query: { enhanced: true } }
  );
</script>

{#each Object.entries(modules) as [path, mod]}
  <enhanced:img src={mod.default} alt={path.split('/').pop()} />
{/each}
```

### Intrinsic dimensions

`<enhanced:img>` auto-sets `width` and `height` from the source. Override with CSS if needed:

```svelte
<style>
  enhanced\:img { width: 100%; height: auto; }
</style>
```

### Per-image transforms

```svelte
<enhanced:img src="./photo.jpg?blur=15" alt="..." />
<enhanced:img src="./photo.jpg?quality=60" alt="..." />
<enhanced:img src="./photo.jpg?rotate=90&flip" alt="..." />
```

Full list: https://github.com/JonasKruckenberg/imagetools/blob/main/docs/directives.md

### CSS selector caveat

Tag-name selectors need the colon escaped:

```svelte
<style>
  enhanced\:img { display: block; }
</style>
```

### Limitations

- SVG only supported statically (not via `?enhanced`)
- Source images must be on your filesystem at build time
- Build output can balloon with many large images

## Dynamic CDN loading

When images aren't available at build time (CMS, DB, user uploads):

### `@unpic/svelte` (CDN-agnostic)

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
  priority
/>
```

Supports Cloudinary, Imgix, Cloudflare Images, Bunny, and many more.

### CMS-bundled

- Contentful has built-in SvelteKit image handling
- Storyblok has an image component
- Contentstack has sample-app integration

### Cloudinary (specific)

```sh
npm i @cloudinary/url-gen
```

```svelte
<script>
  import { Cloudinary } from '@cloudinary/url-gen';
  const cld = new Cloudinary({ cloud: { cloudName: 'demo' } });
  const img = cld.image('sample');
</script>

<img src={img.toURL()} alt="..." />
```

## Vercel image optimization

Configure in adapter:

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-vercel';

export default {
  kit: {
    adapter: adapter({
      images: {
        sizes: [640, 828, 1200, 1920, 3840],
        formats: ['image/avif', 'image/webp'],
        minimumCacheTTL: 300,
        domains: ['my-cms.example.com']
      }
    })
  }
};
```

Use `/\_vercel/image?url=...&w=...&q=...` URLs for runtime transformation.

## Mixing strategies

Different image types benefit from different approaches:

| Image type | Best approach |
|-----------|---------------|
| Hero, brand images | `@sveltejs/enhanced-img` |
| `<meta property="og:image">` | Vite import (don't need responsive) |
| User-uploaded content | CDN with `@unpic/svelte` |
| CMS-provided | `@unpic/svelte` or CMS-native |
| Icons | Inline SVG or Iconify |

## Best practices

- Provide 2x source for HiDPI
- Specify `sizes` for large images
- Set `fetchpriority="high"` for LCP image, omit `loading="lazy"`
- Always provide `alt` text
- Don't change default font-size for `em`/`rem` calculations
- Use `width`/`height` (or CSS aspect-ratio) to prevent CLS
- Serve via CDN for global performance

## See also

- [examples/images.md](../examples/images.md)
- [examples/performance.md](../examples/performance.md)
