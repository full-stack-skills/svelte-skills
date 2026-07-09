# $app/* Modules Examples

The `$app/*` modules give you routing, state, forms, paths, server, environment, and types access.

## $app/forms (legacy form actions)

### 1. enhance — progressive enhancement

```svelte
<script>
  import { enhance } from '$app/forms';
</script>

<form method="POST" action="?/create" use:enhance>
  <input name="title" />
  <button>Create</button>
</form>
```

### 2. enhance with callback

```svelte
<script>
  import { enhance, applyAction } from '$app/forms';
</script>

<form method="POST" action="?/create" use:enhance={() => {
  return async ({ result }) => {
    if (result.type === 'success') {
      await applyAction(result);
    }
  };
}}>
  <!-- ... -->
</form>
```

### 3. deserialize

```js
import { deserialize } from '$app/forms';

async function handleSubmit(event) {
  const response = await fetch('/form?/action', {
    method: 'POST',
    body: new FormData(event.target)
  });
  const result = deserialize(await response.text());
}
```

## $app/navigation

### 4. goto — programmatic navigation

```svelte
<script>
  import { goto } from '$app/navigation';
  function login() {
    goto('/dashboard', { replaceState: true });
  }
</script>
```

Options: `replaceState`, `noScroll`, `keepFocus`, `invalidateAll`, `invalidate`, `state`.

### 5. invalidate / invalidateAll / refreshAll

```svelte
<script>
  import { invalidate, invalidateAll, refreshAll } from '$app/navigation';

  function refreshPost() {
    invalidate('/api/posts/123');  // specific URL
  }
  function refreshAllData() {
    invalidateAll();  // all load functions
  }
  function refreshRemoteAndLoad() {
    refreshAll({ includeLoadFunctions: true });
  }
</script>
```

`refreshAll` re-runs all active remote queries AND optionally load functions.

### 6. Custom invalidate predicate

```js
invalidate((url) => url.pathname === '/api/posts');
```

### 7. preloadCode / preloadData

```svelte
<script>
  import { preloadCode, preloadData } from '$app/navigation';
  onMount(() => {
    preloadCode('/admin');  // import code
    preloadData('/admin');  // also run load
  });
</script>
```

### 8. pushState / replaceState — shallow routing

```svelte
<script>
  import { pushState, replaceState } from '$app/navigation';
  function showModal() {
    pushState('', { showModal: true });
  }
</script>
```

### 9. beforeNavigate / afterNavigate / onNavigate

```svelte
<script>
  import { beforeNavigate, afterNavigate, onNavigate } from '$app/navigation';

  beforeNavigate(({ cancel, type }) => {
    if (type === 'leave' && dirty) {
      if (!confirm('Leave without saving?')) cancel();
    }
  });

  afterNavigate(({ to }) => {
    console.log('navigated to', to.url.pathname);
  });

  onNavigate((navigation) => {
    return new Promise((resolve) => {
      document.startViewTransition(async () => {
        resolve();
        await navigation.complete;
      });
    });
  });
</script>
```

`onNavigate` callback must run during component initialization; can return a Promise or a function to run after DOM update.

### 10. disableScrollHandling

```svelte
<script>
  import { disableScrollHandling } from '$app/navigation';
  afterNavigate(() => disableScrollHandling());  // generally discouraged
</script>
```

## $app/state

### 11. page state

```svelte
<script>
  import { page } from '$app/state';
  const user = $derived(page.data.user);
  const id = $derived(page.params.id);
</script>

<p>Currently at {page.url.pathname}</p>
{#if page.error}
  <p>Error: {page.error.message}</p>
{/if}
```

`page` is reactive; use it inside `$derived` or templates.

### 12. navigating

```svelte
<script>
  import { navigating } from '$app/state';
</script>

{#if navigating.to}
  <p>Navigating to {navigating.to.url.pathname}...</p>
{/if}
```

`navigating` is `null` when no navigation, during SSR, and during prerender.

### 13. updated — version check

```svelte
<script>
  import { updated } from '$app/state';
</script>

{#if updated.current}
  <button onclick={() => location.reload()}>Reload for new version</button>
{/if}
```

`updated.check()` forces an immediate check.

## $app/paths

### 14. asset — resolve static files

```svelte
<script>
  import { asset } from '$app/paths';
</script>

<img src={asset('/favicon.png')} alt="logo" />
```

`asset()` handles the base path automatically. `Asset` is a union of static files + wildcard.

### 15. resolve — pathnames and route IDs

```js
import { resolve } from '$app/paths';

const path = resolve('/blog/hello-world');                       // pathname
const other = resolve('/blog/[slug]', { slug: 'hello-world' }); // route ID + params
```

### 16. match — match path to route ID

```js
import { match } from '$app/paths';

const route = await match('/blog/hello-world');
if (route?.id === '/blog/[slug]') {
  const slug = route.params.slug;
}
```

## $app/server

### 17. read — read imported assets

```js
import { read } from '$app/server';
import somefile from './somefile.txt';

export async function GET() {
  const asset = read(somefile);
  const text = await asset.text();
  return new Response(text);
}
```

Server-only.

### 18. getRequestEvent — request event in remote functions

```js
import { getRequestEvent, query } from '$app/server';

export const getProfile = query(async () => {
  const { cookies, locals } = getRequestEvent();
  return { user: locals.user, session: cookies.get('session_id') };
});
```

## $app/environment

### 19. browser / building / dev / version

```js
import { browser, building, dev, version } from '$app/environment';

if (browser) console.log('client side');
if (building) console.log('vite build running');
if (dev) console.log('dev server');
console.log(version);
```

## $app/types

### 20. RouteId / Pathname / RouteParams

```ts
import type { RouteId, Pathname, RouteParams } from '$app/types';

const id: RouteId = '/blog/[slug]';
const path: Pathname = '/blog/hello';
type BlogParams = RouteParams<'/blog/[slug]'>;  // { slug: string }
```

These are generated from your routes.

## $lib alias

### 21. Use $lib

```svelte
<!-- src/lib/Component.svelte -->
```

```svelte
<!-- src/routes/+page.svelte -->
<script>
  import Component from '$lib/Component.svelte';
</script>
<Component />
```

Anything under `src/lib/` is accessible via `$lib`.

## $service-worker

### 22. Inside service worker

```js
import { base, build, files, prerendered, version } from '$service-worker';
```