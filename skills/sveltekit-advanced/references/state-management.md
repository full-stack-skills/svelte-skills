# State Management Reference

## Core principles

- Server is stateless — module-level variables leak between requests.
- `load` functions must be pure — no global store writes, no side effects.
- `app state` and `app stores` use Svelte's context API on the server.

## Where state can live

| Location | Survives reload | Survives nav | Affects SSR | Per-user safe |
|----------|-----------------|--------------|-------------|---------------|
| Module-level (BAD) | No | No | Yes | NO — leaks |
| Cookies | Yes | No | Yes | Yes |
| Database | Yes | Yes | Yes | Yes |
| URL search params | Yes | Yes | Yes | Yes |
| `$state` in component | No | Yes (per component) | N/A | Yes |
| Context (`setContext`) | No | No | Yes | Yes |
| Snapshot | No | Yes (history-bound) | N/A | Yes |
| Service worker | Yes | Yes | N/A | Mostly |

## Context API rules

```js
setContext('key', value);
const v = getContext('key');
```

- Pass a FUNCTION into `setContext` to keep reactivity across boundaries:
  ```js
  setContext('user', () => data.user);  // good
  setContext('user', data.user);        // freezes value
  ```
- Context updates during SSR do NOT propagate up — child renders after parent.
- Pass state down as props to avoid hydration flash.

## Component lifecycle on navigation

- Existing components are REUSED when navigating between routes that share layout.
- `const derived = data.x` computes once and never updates.
- Use `$derived(data.x)` for values that should recompute.
- `onMount`/`onDestroy` do NOT re-run on nav.

```svelte
<script>
  import { page } from '$app/state';
  let { data } = $props();

  let wordCount = $derived(data.content.split(' ').length);
  let readingTime = $derived(wordCount / 250);
</script>
```

## Force remount

```svelte
{#key page.url.pathname}
  <BlogPost {data} />
{/key}
```

## Lifecycle hooks

```svelte
import { afterNavigate, beforeNavigate, onNavigate } from '$app/navigation';
```

- `beforeNavigate({ cancel, type, from, to })` — run before nav; `cancel()` blocks.
- `afterNavigate(({ from, to }) => ...)` — run after nav completes.
- `onNavigate(...)` — same as `beforeNavigate` but allows returning a Promise (e.g., `document.startViewTransition`).

## URL state

```svelte
<script>
  import { page } from '$app/state';
  import { goto } from '$app/navigation';

  let sort = $derived(page.url.searchParams.get('sort') ?? 'name');

  function setSort(field) {
    const url = new URL(page.url);
    url.searchParams.set('sort', field);
    goto(url, { keepFocus: true, noScroll: true });
  }
</script>
```

## Snapshots

```ts
export const snapshot: Snapshot<T> = {
  capture: () => T,
  restore: (value: T) => void
};
```

- Stored in `sessionStorage`.
- JSON-serializable only.
- Captured before page updates.
- Restored on history nav.

## What NOT to do

```js
// BAD: global server state
let user;
export function load() { return { user }; }

// BAD: side effects in load
export async function load({ fetch }) {
  user.set(await fetch('/api/user').then((r) => r.json()));
}

// BAD: const derived from data
const wordCount = data.content.split(' ').length;
```

## page.data and page.state

- `page.data` — merged data from all active `load` functions.
- `page.state` — set via `pushState`/`replaceState`/`goto(..., { state })`. Always `{}` on SSR and first paint.

## $app/stores (legacy)

Deprecated since SvelteKit 2.12. Use `$app/state` with Svelte 5 runes.