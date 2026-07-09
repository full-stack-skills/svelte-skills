# Link Options Reference

Attributes on `<a>` and `<form method="GET">` (or any ancestor). SvelteKit reads them when the link is rendered.

## Attributes

### data-sveltekit-preload-data

| Value | Behavior |
|-------|----------|
| `hover` (default) | Preload on hover (desktop) or `touchstart` (mobile) |
| `tap` | Preload on `touchstart` or `mousedown` |

Skipped when `navigator.connection.saveData === true`.

### data-sveltekit-preload-code

| Value | Behavior |
|-------|----------|
| `eager` | Preload straight away |
| `viewport` | Preload when in viewport |
| `hover` | Preload on hover/touchstart |
| `tap` | Preload on tap/click |

`eager`/`viewport` apply only to links in the DOM immediately after navigation. Won't auto-attach to late-added links.

Preloading code is a prerequisite for data, so this attribute only takes effect if more eager than `data-sveltekit-preload-data`.

### data-sveltekit-reload

Forces full-page navigation. Equivalent behavior to `rel="external"` (which gets this treatment automatically).

```html
<a data-sveltekit-reload href="/legacy">Legacy</a>
```

Also: prerendering skips links with `data-sveltekit-reload` or `rel="external"`.

### data-sveltekit-replacestate

Replaces the current history entry instead of pushing a new one.

```html
<a data-sveltekit-replacestate href="?filter=x">Filter</a>
```

Programmatic equivalent: `goto(url, { replaceState: true })`.

### data-sveltekit-keepfocus

Keeps the currently-focused element focused after navigation.

```html
<form data-sveltekit-keepfocus method="GET">
  <input name="q" autofocus />
</form>
```

Avoid on `<a>` — the focused element is the anchor itself. Only use on elements that still exist after navigation.

### data-sveltekit-noscroll

Disables scroll-to-top after navigation (default behavior).

```html
<a data-sveltekit-noscroll href="/modal">Open</a>
```

## Disabling options in a subtree

```html
<div data-sveltekit-preload-data>
  <a href="/a">preloaded</a>
  <div data-sveltekit-preload-data="false">
    <a href="/d">NOT preloaded</a>
  </div>
</div>
```

## Conditional application (Svelte)

```svelte
<div data-sveltekit-preload-data={condition ? 'hover' : false}>
```

## Programmatic preloading

```ts
import { preloadCode, preloadData } from '$app/navigation';

preloadCode('/admin');   // import code only
preloadData('/admin');   // import code + run load
```

`preloadData` returns `{ type: 'loaded', status, data } | { type: 'redirect', location }`.

## Forms

All attributes (except `replacestate`) work on `<form method="GET">`. Same rules apply.

## Save-Data header

When user has `navigator.connection.saveData === true`:
- `preload-data` and `preload-code` are ignored.

## Click modifiers (shallow routing)

```svelte
<a
  href="/photos/{id}"
  onclick={async (e) => {
    if (e.shiftKey || e.metaKey || e.ctrlKey || innerWidth < 640) return;
    e.preventDefault();
    const result = await preloadData(href);
    if (result.type === 'loaded') pushState(href, { selected: result.data });
  }}
>
```

## Defaults

The default SvelteKit template includes `data-sveltekit-preload-data="hover"` on `<body>` in `app.html`. Override per-link or per-subtree.

## Caching behavior

Preload only fires the `fetch` for `load`; the response is cached for the actual navigation if it happens shortly after.

## Combining

```html
<a
  data-sveltekit-preload-code="viewport"
  data-sveltekit-preload-data="hover"
  data-sveltekit-keepfocus
  data-sveltekit-noscroll
  href="/dashboard"
>Dashboard</a>
```

`preload-code` more eager than `preload-data` → code preloads earlier than data.