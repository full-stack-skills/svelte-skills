# Link Options Examples

`data-sveltekit-*` attributes on `<a>` and `<form method="GET">` elements.

## 1. Default preload on hover (set in app.html)

```html
<!-- src/app.html -->
<body data-sveltekit-preload-data="hover">
  <div style="display: contents">%sveltekit.body%</div>
</body>
```

Every link preloads `load` data when hovered.

## 2. tap instead of hover for volatile data

```html
<a data-sveltekit-preload-data="tap" href="/stonks">
  Get current stonk values
</a>
```

Preload only when tapped/clicked (avoids stale preloads).

## 3. Preload code without data

```html
<a data-sveltekit-preload-code="viewport" href="/about">
  About (code preloaded when in viewport)
</a>
```

Values: `eager`, `viewport`, `hover`, `tap`. `viewport`/`eager` only apply to links present at navigation time.

## 4. Disable options for a subtree

```html
<div data-sveltekit-preload-data>
  <a href="/a">preloaded</a>
  <div data-sveltekit-preload-data="false">
    <a href="/d">NOT preloaded</a>
  </div>
</div>
```

## 5. Conditional in Svelte template

```svelte
<div data-sveltekit-preload-data={isLoggedIn ? 'hover' : false}>
  <!-- links preloaded only when logged in -->
</div>
```

## 6. Force full-page reload

```html
<a data-sveltekit-reload href="/path">Path</a>
```

Also applied automatically to `<a rel="external">`. Excluded from prerendering.

## 7. Replace history entry instead of push

```html
<a data-sveltekit-replacestate href="/filter">Filter</a>
```

Useful for filter changes that shouldn't pollute history.

## 8. Keep focus on search input

```html
<form data-sveltekit-keepfocus method="GET">
  <input type="text" name="query" autofocus />
</form>
```

After submission, focus stays on the input.

## 9. Disable scroll-to-top

```html
<a href="/modal" data-sveltekit-noscroll>Open modal</a>
```

Useful for in-page modals.

## 10. Combine multiple options

```html
<a
  data-sveltekit-preload-data="tap"
  data-sveltekit-preload-code="hover"
  data-sveltekit-keepfocus
  href="/dynamic"
>
  Dynamic page
</a>
```

## 11. Programmatic preloadData

```svelte
<script>
  import { preloadData } from '$app/navigation';
  function handleHover() {
    preloadData('/dashboard');
  }
</script>

<a href="/dashboard" onmouseenter={handleHover}>Dashboard</a>
```

## 12. Apply to a parent of links

```html
<nav data-sveltekit-preload-data="hover">
  <a href="/home">Home</a>
  <a href="/about">About</a>
  <a href="/contact">Contact</a>
</nav>
```

## 13. data-sveltekit-replacestate via goto

```svelte
<script>
  import { goto } from '$app/navigation';
  function applyFilter(filter) {
    goto(`?filter=${filter}`, { replaceState: true });
  }
</script>
```

## 14. preloadCode programmatic

```svelte
<script>
  import { preloadCode } from '$app/navigation';
  onMount(() => {
    preloadCode('/settings');  // warm up the route
  });
</script>
```

## 15. Respect Save-Data

All preload attributes honor `navigator.connection.saveData === true` — data is never preloaded.

## 16. Form GET preload

```html
<form data-sveltekit-preload-data="hover" method="GET" action="/search">
  <input name="q" />
</form>
```

Preloads the search results page on hover.

## 17. Link with hash (scroll behavior)

```html
<a href="/docs#section-3">Section 3</a>
```

Default: scrolls to element with id `section-3`. Add `data-sveltekit-noscroll` to disable.

## 18. Preload on viewport via IntersectionObserver

```html
<a data-sveltekit-preload-code="viewport" href="/heavy-route">
  Heavy page
</a>
```

Only works for links in the DOM immediately after navigation. Late-added links need `hover`/`tap`.