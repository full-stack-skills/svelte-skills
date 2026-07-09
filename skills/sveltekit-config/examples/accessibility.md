# Accessibility

SvelteKit provides accessible defaults. You own the rest.

## 1. Set the `<title>` on every page

SvelteKit injects a live region that announces the new page title after each navigation.

```svelte
<!-- src/routes/+page.svelte -->
<svelte:head>
  <title>Todo List — MyApp</title>
</svelte:head>
```

Make titles unique and descriptive. Same for SEO.

## 2. Set the `lang` attribute

```html
<!-- src/app.html -->
<html lang="en">
```

For multi-language sites:

```html
<!-- src/app.html -->
<html lang="%lang%">
```

```js
// src/hooks.server.js
import { get_lang } from '$lib/i18n';

export async function handle({ event, resolve }) {
  return resolve(event, {
    transformPageChunk: ({ html }) => html.replace('%lang%', get_lang(event))
  });
}
```

## 3. Focus management — `afterNavigate`

By default SvelteKit focuses `<body>` after navigation. Override for specific pages:

```js
// src/routes/form/+page.js
import { afterNavigate } from '$app/navigation';

afterNavigate(() => {
  document.querySelector('h1')?.focus();
});
```

## 4. Preserve focus on `<form>` submission

```svelte
<form method="POST" data-sveltekit-keepfocus>
  <input name="query" />
</form>
```

The currently-focused input keeps focus after enhanced submission.

## 5. Preserve focus on programmatic nav

```js
import { goto } from '$app/navigation';

await goto('/results', { keepFocus: true });
```

Only do this if the focused element still exists after navigation — otherwise assistive tech users lose their place.

## 6. Skip-to-content link

```svelte
<!-- src/routes/+layout.svelte -->
<a href="#main" class="skip-link">Skip to main content</a>

<main id="main" tabindex="-1">
  {@render children()}
</main>
```

```svelte
<style>
  .skip-link {
    position: absolute;
    top: -40px;
    left: 0;
    background: #000;
    color: #fff;
    padding: 8px;
  }
  .skip-link:focus {
    top: 0;
  }
</style>
```

## 7. Disable `autofocus` carefully

SvelteKit honors `autofocus` if present:

```svelte
<input autofocus />
```

But autofocus is disruptive for screen reader users — only use on search pages or login screens.

## 8. Headings hierarchy

Each page should have exactly one `<h1>`, then `<h2>` for sections, etc.

```svelte
<h1>Articles</h1>
<section>
  <h2>Latest</h2>
  <article>
    <h3>Title</h3>  <!-- ← h3, not h1, not h2 -->
  </article>
</section>
```

## 9. Semantic HTML

```svelte
<!-- BAD -->
<div onclick={handleClick}>Click me</div>

<!-- GOOD -->
<button onclick={handleClick}>Click me</button>
```

Svelte's compiler warns on `noninteractive_click_interactive` etc.

## 10. ARIA only when needed

Native HTML is best. Add ARIA only when native elements can't express the pattern.

```svelte
<!-- Only when no native alternative -->
<button aria-expanded={isOpen} aria-controls="menu">Menu</button>
<ul id="menu" role="menu" hidden={!isOpen}>...</ul>
```

## See also

- [a11y reference](../references/a11y-reference.md)
- [Svelte a11y warnings](https://svelte.dev/docs/svelte/compiler-warnings)
