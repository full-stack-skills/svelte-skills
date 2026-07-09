# Snapshots & Shallow Routing Examples

Preserve DOM state on back-nav and create history entries without navigating.

## Snapshots

### 1. Snapshot a single string field

```svelte
<!-- src/routes/comments/+page.svelte -->
<script>
  let comment = $state('');

  /** @type {import('./$types').Snapshot<string>} */
  export const snapshot = {
    capture: () => comment,
    restore: (value) => (comment = value)
  };
</script>

<form method="POST">
  <textarea bind:value={comment}></textarea>
  <button>Post</button>
</form>
```

`capture` runs before navigating away; `restore` runs when navigating back.

### 2. Snapshot multiple fields

```svelte
<script>
  let title = $state('');
  let body = $state('');
  let draft = $state({ saved: false });

  /** @type {import('./$types').Snapshot<{ title: string; body: string; saved: boolean }>} */
  export const snapshot = {
    capture: () => ({ title, body, saved: draft.saved }),
    restore: (value) => {
      title = value.title;
      body = value.body;
      draft.saved = value.saved;
    }
  };
</script>
```

### 3. Snapshot in +layout.svelte

```svelte
<!-- src/routes/admin/+layout.svelte -->
<script>
  let sidebarOpen = $state(true);
  let selectedTab = $state('overview');

  export const snapshot = {
    capture: () => ({ sidebarOpen, selectedTab }),
    restore: (value) => {
      sidebarOpen = value.sidebarOpen;
      selectedTab = value.selectedTab;
    }
  };
</script>
```

Snapshots work in layouts too — they apply to all child pages.

### 4. JSON-only constraint

Snapshots are stored in `sessionStorage`, so they must be JSON-serializable. No `Date`, no functions, no circular references.

### 5. Don't return huge objects

The capture value is held in memory for the whole session. Avoid storing large arrays.

## Shallow routing

### 6. Basic modal

```svelte
<!-- src/routes/+page.svelte -->
<script>
  import { pushState } from '$app/navigation';
  import { page } from '$app/state';
  import Modal from './Modal.svelte';

  function showModal() {
    pushState('', { showModal: true });
  }
</script>

<button onclick={showModal}>Open modal</button>

{#if page.state.showModal}
  <Modal close={() => history.back()} />
{/if}
```

Back button dismisses the modal.

### 7. Type-safe page state

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface PageState {
      showModal?: boolean;
      selected?: { id: string; title: string };
    }
  }
}
export {};
```

Now `page.state` is typed.

### 8. replaceState — no new history entry

```svelte
<script>
  import { replaceState } from '$app/navigation';
  import { page } from '$app/state';

  function toggleTheme() {
    replaceState(page.url, { ...page.state, dark: !page.state.dark });
  }
</script>
```

### 9. Photo gallery with shallow route

```svelte
<!-- src/routes/photos/+page.svelte -->
<script>
  import { preloadData, pushState, goto } from '$app/navigation';
  import { page } from '$app/state';
  import Modal from './Modal.svelte';
  import PhotoPage from './[id]/+page.svelte';

  let { data } = $props();
</script>

{#each data.thumbnails as thumbnail}
  <a
    href="/photos/{thumbnail.id}"
    onclick={async (e) => {
      if (innerWidth < 640 || e.shiftKey || e.metaKey || e.ctrlKey) return;
      e.preventDefault();

      const { href } = e.currentTarget;
      const result = await preloadData(href);

      if (result.type === 'loaded' && result.status === 200) {
        pushState(href, { selected: result.data });
      } else {
        goto(href);
      }
    }}
  >
    <img alt={thumbnail.alt} src={thumbnail.src} />
  </a>
{/each}

{#if page.state.selected}
  <Modal onclose={() => history.back()}>
    <PhotoPage data={page.state.selected} />
  </Modal>
{/if}
```

### 10. Shallow routing caveats

```svelte
<!-- page.state is always {} on SSR and first paint -->
<!-- onMount runs after first render, by then state may be populated -->
<script>
  import { onMount } from 'svelte';
  import { page } from '$app/state';

  let mounted = $state(false);
  onMount(() => (mounted = true));
</script>

{#if mounted && page.state.showModal}
  <Modal />
{/if}
```

### 11. Open modal from URL on direct visit (fallback)

```svelte
<script>
  import { page } from '$app/state';
  import { onMount } from 'svelte';

  let isModalOpen = $state(false);

  onMount(() => {
    if (page.url.searchParams.has('modal')) {
      isModalOpen = true;
    }
  });
</script>
```

Shallow routing requires JS — provide a real URL fallback.

### 12. Multiple modals with discriminated state

```svelte
<script>
  import { pushState } from '$app/navigation';
  import { page } from '$app/state';

  function openModal(name, data) {
    pushState('', { modal: name, modalData: data });
  }
</script>

{#if page.state.modal === 'edit'}
  <EditModal data={page.state.modalData} />
{:else if page.state.modal === 'confirm'}
  <ConfirmModal data={page.state.modalData} />
{/if}
```

### 13. Snapshot + shallow routing together

Snapshots apply to the current page; shallow routing creates new history entries. They compose — back-nav can both restore a snapshot AND clear `page.state`.

### 14. Disable scroll on shallow route

```js
pushState('', { showModal: true });
// scroll doesn't reset because no real navigation happens
```

Useful for modals that shouldn't scroll the page.

### 15. Replace vs push for filters

```js
// Use replaceState for filter changes (don't pollute history)
function applyFilter(f) {
  replaceState(`?filter=${f}`, {});
}

// Use pushState for modal open (so back button closes modal)
function openModal() {
  pushState('', { showModal: true });
}
```