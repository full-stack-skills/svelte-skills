# Accessibility reference

SvelteKit provides accessible defaults. This covers what SvelteKit handles automatically and what you need to do yourself.

## What SvelteKit handles

### Live region for route announcements

SvelteKit injects an ARIA live region that reads the new `<title>` after each client-side navigation. This simulates the "page title announced" behavior of full-page reloads.

The page name is taken from the `<title>` element. Every page must have one.

### Focus management

After each navigation, SvelteKit focuses the `<body>` element. Exception: if `[autofocus]` is present, that element is focused instead.

### `<svelte:head>` support

The `<svelte:head>` element lets each page declare `<title>`, `<meta>`, and other head content — works the same as native HTML.

### Reduced motion / data preferences

SvelteKit respects:
- `prefers-reduced-motion` (affects preloading heuristics)
- `navigator.connection.saveData` (disables preloading)

### Build-time a11y checks

Svelte 5's compiler emits warnings for many a11y issues:
- Missing `alt` on `<img>`
- `click` on non-interactive elements
- Missing labels on form controls
- `tabindex` > 0 (anti-pattern)

## What you must do

### 1. Unique, descriptive page titles

```svelte
<svelte:head>
  <title>{data.title} — MyApp</title>
</svelte:head>
```

### 2. Set the `lang` attribute

```html
<!-- src/app.html -->
<html lang="en">
```

For multi-language sites, use a `transformPageChunk`:

```js
return resolve(event, {
  transformPageChunk: ({ html }) => html.replace('%lang%', getLang(event))
});
```

### 3. Semantic HTML

```svelte
<!-- BAD -->
<div onclick={open}>Open menu</div>

<!-- GOOD -->
<button onclick={open}>Open menu</button>
```

### 4. Labels for form controls

```svelte
<label>
  Email
  <input type="email" name="email" />
</label>

<!-- or -->
<label for="email">Email</label>
<input id="email" type="email" name="email" />
```

### 5. Skip-to-content link

```svelte
<a href="#main" class="skip-link">Skip to content</a>
<main id="main" tabindex="-1">{@render children()}</main>
```

### 6. ARIA only when necessary

Native HTML is preferred. Add ARIA only when the pattern can't be expressed in HTML:

```svelte
<button aria-expanded={open} aria-controls="menu">Menu</button>
<ul id="menu" hidden={!open}>{#each items as item}<li>{item}</li>{/each}</ul>
```

## Focus management

### Custom focus on navigation

```js
// src/routes/form/+page.js
import { afterNavigate } from '$app/navigation';

afterNavigate(() => {
  document.querySelector('h1')?.focus();
});
```

### Preserve focus on form submission

```svelte
<form method="POST" data-sveltekit-keepfocus>
  <input name="q" />
</form>
```

### Preserve focus on programmatic nav

```js
import { goto } from '$app/navigation';
await goto('/results', { keepFocus: true });
```

Only use if the focused element still exists after navigation.

## Keyboard navigation

### Skip links

See above.

### Focus traps (modals)

```svelte
<script>
  import { onMount, onDestroy } from 'svelte';
  let dialog;
  let firstFocusable;
  
  onMount(() => {
    firstFocusable = dialog.querySelector('button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])');
    firstFocusable?.focus();
  });
</script>

<div bind:this={dialog} role="dialog" aria-modal="true" aria-labelledby="title">
  <h2 id="title">Confirm</h2>
  <button>Cancel</button>
  <button>OK</button>
</div>
```

Or use a library like `focus-trap-svelte`.

## Live regions

### Polite (most common)

```svelte
<div aria-live="polite" aria-atomic="true">
  {statusMessage}
</div>
```

Use for non-urgent updates (form submission results, search results loaded).

### Assertive

```svelte
<div aria-live="assertive" role="alert">
  {errorMessage}
</div>
```

Use only for urgent updates (validation errors that block action).

## Color contrast

WCAG AA minimums:
- 4.5:1 for body text
- 3:1 for large text (18pt+ or 14pt bold)
- 3:1 for UI components and graphics

Tools:
- Lighthouse audit
- axe DevTools browser extension
- Stark (Figma plugin)

## Screen reader testing

- **VoiceOver** (macOS, iOS — `CMD+F5`)
- **NVDA** (Windows — free)
- **JAWS** (Windows — paid)
- **TalkBack** (Android)
- **Orca** (Linux)

## Resources

- [MDN Web Docs: Accessibility](https://developer.mozilla.org/en-US/docs/Learn/Accessibility)
- [The A11y Project](https://www.a11yproject.com/)
- [WCAG Quick Reference](https://www.w3.org/WAI/WCAG21/quickref/)
- [Svelte a11y warnings](https://svelte.dev/docs/svelte/compiler-warnings)

## See also

- [examples/accessibility.md](../examples/accessibility.md)
