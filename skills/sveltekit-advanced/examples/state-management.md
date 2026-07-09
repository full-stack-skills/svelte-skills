# State Management Examples

SSR-safe patterns for managing state across server and client in SvelteKit.

## 1. Never store user data in module-level server variables

```js
// src/routes/+page.server.js
// WRONG — leaks between users
let user;
export function load() { return { user }; }

// RIGHT — auth + DB
import * as db from '$lib/server/database';
export async function load({ cookies }) {
  const session = cookies.get('session');
  return { user: await db.getUserBySession(session) };
}
```

## 2. Pass state down via context (SSR-safe)

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import { setContext } from 'svelte';
  let { data } = $props();
  // Pass a FUNCTION, not the value, so reactivity crosses boundaries
  setContext('user', () => data.user);
</script>

<slot />
```

```svelte
<!-- src/routes/profile/+page.svelte -->
<script>
  import { getContext } from 'svelte';
  const user = getContext('user');
</script>

<h1>Welcome {user().name}</h1>
```

## 3. URL search params for shareable state

```svelte
<!-- src/routes/products/+page.svelte -->
<script>
  import { page } from '$app/state';
  import { goto } from '$app/navigation';

  let sort = $derived(page.url.searchParams.get('sort') ?? 'name');
  let order = $derived(page.url.searchParams.get('order') ?? 'asc');

  function setSort(field) {
    const url = new URL(page.url);
    url.searchParams.set('sort', field);
    goto(url, { keepFocus: true, noScroll: true });
  }
</script>

<select onchange={(e) => setSort(e.target.value)}>
  <option value="name">Name</option>
  <option value="price">Price</option>
</select>
```

```js
// src/routes/products/+page.server.js
export async function load({ url }) {
  const sort = url.searchParams.get('sort') ?? 'name';
  const order = url.searchParams.get('order') ?? 'asc';
  return { products: await db.getProducts({ sort, order }) };
}
```

## 4. Component state preserved across navigation

```svelte
<!-- src/routes/blog/[slug]/+page.svelte -->
<script>
  let { data } = $props();

  // WRONG — only computed once on mount
  // const wordCount = data.content.split(' ').length;

  // RIGHT — recomputes when data changes
  let wordCount = $derived(data.content.split(' ').length);
  let readingTime = $derived(Math.ceil(wordCount / 250));
</script>

<p>Reading time: {readingTime} minutes</p>
```

## 5. Force remount on navigation

```svelte
<script>
  import { page } from '$app/state';
</script>

{#key page.url.pathname}
  <BlogPost data={data} />
{/key}
```

## 6. Snapshots — preserve form drafts

```svelte
<!-- src/routes/comments/+page.svelte -->
<script>
  let comment = $state('');

  export const snapshot = {
    capture: () => comment,
    restore: (value) => (comment = value)
  };
</script>

<textarea bind:value={comment}></textarea>
```

`capture` is called before navigating away. `restore` is called on back-navigation. Data is JSON-serializable; stored in `sessionStorage`.

## 7. Re-run lifecycle on navigation

```svelte
<script>
  import { afterNavigate, beforeNavigate } from '$app/navigation';

  afterNavigate(() => console.log('navigated to', $page.url.pathname));
  beforeNavigate(({ cancel }) => {
    if (hasUnsavedChanges) {
      cancel();
      showConfirmDialog();
    }
  });
</script>
```

## 8. Stores (legacy — prefer $state in Svelte 5)

```js
// src/lib/stores/cart.js
import { writable } from 'svelte/store';

export const cart = writable({
  items: [],
  add(item) {
    cart.update((c) => ({ ...c, items: [...c.items, item] }));
  }
});
```

```svelte
<script>
  import { cart } from '$lib/stores/cart';
  import { getContext } from 'svelte';

  // SSR-safe: get from context in +layout
  const cart = getContext('cart');
</script>
```

## 9. Pure load function

```js
// +page.server.js — no side effects!
export async function load({ params, fetch }) {
  const post = await fetch(`/api/posts/${params.slug}`).then((r) => r.json());
  return { post };  // just return data
}
```

## 10. Auth via cookies (recommended over module state)

```js
// hooks.server.js
export async function handle({ event, resolve }) {
  const session = event.cookies.get('session');
  event.locals.user = session ? await db.getUserBySession(session) : null;
  return resolve(event);
}
```

## 11. Global state without context (client-only SPA)

```js
// src/lib/state.js
// ONLY safe in client-only apps without SSR
export const globalState = $state({ count: 0 });
```

```svelte
<script>
  import { globalState } from '$lib/state';
</script>
<button onclick={() => globalState.count++}>{globalState.count}</button>
```

## 12. Pass props down instead of mutating context up

```svelte
<!-- Parent: pass the state down -->
<Child items={items} />

<!-- Child: emit events up -->
<Child onAdd={(item) => items.push(item)} />
```

Avoid `setContext('items', v => v.push(newItem))` — SSR parent has already rendered.
