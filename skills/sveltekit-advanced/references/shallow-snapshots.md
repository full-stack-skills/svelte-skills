# Snapshots & Shallow Routing Reference

## Snapshots

### API

```ts
import type { Snapshot } from './$types';

export const snapshot: Snapshot<T> = {
  capture: () => T,           // called before nav away
  restore: (value: T) => void  // called on history-nav back
};
```

Export from `+page.svelte` or `+layout.svelte`.

### Storage

- Saved to `sessionStorage`.
- Must be JSON-serializable (no Date, Map, Set, functions, cycles).
- Persists across reloads.
- Held in memory for the session.

### When called

- `capture`: immediately before page updates on navigation away.
- `restore`: when the page updates on history navigation (back/forward).

### Caveats

- Don't return large objects.
- Avoid storing transient state that's better in URL or component.
- Snapshot data is shared across tabs in the same session.

### Layout snapshots

Layout snapshots apply to ALL child pages' navigation. Captured once per route tree change.

## Shallow routing

### API

```ts
import { pushState, replaceState } from '$app/navigation';

pushState(url: string | URL, state: App.PageState): void
replaceState(url: string | URL, state: App.PageState): void
```

- `url: ''` keeps current URL.
- `state` becomes `page.state` (and `App.PageState` if declared).

### Reading state

```svelte
<script>
  import { page } from '$app/state';
</script>

{#if page.state.showModal}
  <Modal />
{/if}
```

### Type the state

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface PageState {
      showModal?: boolean;
      selected?: { id: string; title: string };
      filter?: string;
    }
  }
}
export {};
```

### replaceState vs pushState

- `pushState` — creates a new history entry (back button goes to previous page).
- `replaceState` — replaces current entry (back button skips this state).

### Loading data for shallow route

```svelte
<a
  href="/photos/{id}"
  onclick={async (e) => {
    if (e.shiftKey || e.metaKey || e.ctrlKey || innerWidth < 640) return;
    e.preventDefault();
    const result = await preloadData(e.currentTarget.href);
    if (result.type === 'loaded' && result.status === 200) {
      pushState(e.currentTarget.href, { selected: result.data });
    } else if (result.type === 'redirect') {
      goto(result.location);
    }
  }}
>
  <img src={thumbnail.src} alt={thumbnail.alt} />
</a>
```

`preloadData` reuses an in-flight preload triggered by `data-sveltekit-preload-data`.

### Rendering another page in a modal

```svelte
{#if page.state.selected}
  <Modal onclose={() => history.back()}>
    <PhotoPage data={page.state.selected} />
  </Modal>
{/if}
```

Pass the loaded data as `data` prop to the child `+page.svelte`.

### Caveats

- `page.state` is `{}` during SSR and first paint.
- Requires JavaScript.
- Provide a real URL fallback (don't rely on `page.state` for content accessible without JS).
- Browser back button works as expected.
- Doesn't re-run load functions for the new "route".
- Does NOT change what's in `page.url` for rendering purposes (it only affects `page.state`).

### goto with state

```ts
goto('/dashboard', { state: { tab: 'settings' } });
```

State accessible via `page.state.tab`.

### Use cases

- Modals (back button dismisses).
- Drawers/sheets.
- Photo lightboxes.
- Tab navigation without re-rendering.
- Multi-step wizards that share URL but track progress in state.

### State persistence

Shallow route state does NOT survive reload — it's part of `history.state` which is recreated on page load. Use snapshots if you want reload-survival.

### Pattern: modal with URL fallback

```svelte
<script>
  import { page } from '$app/state';
  import { onMount } from 'svelte';

  let open = $state(false);
  onMount(() => {
    if (page.url.searchParams.has('modal')) open = true;
  });
</script>

{#if open}
  <Modal onclose={() => { open = false; history.replaceState(null, '', page.url.pathname); }} />
{/if}
```

This works without JS (URL has the param) and without state (URL has the param).

### Don't abuse shallow routing

- For full-page navigation, use `<a href>`.
- For stateful UI without history, use `$state`.
- For URL-shareable state, use search params.
- For session-spanning UI, use snapshots.