# svelte:boundary Examples

`<svelte:boundary>` (added in Svelte 5.3.0) lets you "wall off" parts of your app so that you can:

- provide UI for when `await` expressions inside the boundary are still resolving (`pending` snippet), and
- handle errors that occur during rendering or while running effects (`failed` snippet / `onerror` handler).

These examples show every property, the `transformError` SSR pattern, and common real-world patterns.

---

## 1. Basic `pending` Snippet (await loading)

The `pending` snippet is shown until every `await` expression inside the boundary has resolved. It is **only** shown for the initial render — for subsequent async updates, use [`$effect.pending()`](https://svelte.dev/docs/svelte/$effect#$effect.pending).

```svelte
<script>
  async function delayed(message) {
    await new Promise((r) => setTimeout(r, 1500));
    return message;
  }
</script>

<svelte:boundary>
  <p>{await delayed('hello!')}</p>

  {#snippet pending()}
    <p>loading...</p>
  {/snippet}
</svelte:boundary>
```

---

## 2. Basic `failed` Snippet with `reset`

The `failed` snippet receives the `error` and a `reset` function that recreates the boundary's content (clears the error and re-runs children).

```svelte
<svelte:boundary>
  <FlakyComponent />

  {#snippet failed(error, reset)}
    <div class="error-state">
      <p>Something went wrong: {error.message}</p>
      <button onclick={reset}>oops! try again</button>
    </div>
  {/snippet}
</svelte:boundary>
```

The `failed` snippet is the only thing rendered after a handled error — the boundary's existing content is removed.

---

## 3. Passing the `failed` Snippet as a Property

Snippets can be passed **explicitly** as a property, just like any other component snippet prop:

```svelte
<script>
  import MyErrorUI from './MyErrorUI.svelte';
</script>

<svelte:boundary failed={MyErrorUI}>
  <FlakyComponent />
</svelte:boundary>
```

…or **implicitly** by declaring it directly inside the boundary (as in example 2). Both forms are equivalent.

---

## 4. `onerror` Handler for Error Reporting

`onerror` receives the same `error` and `reset` arguments as `failed`. The most common use is forwarding to an error reporting service (Sentry, Bugsnag, etc.) without breaking the UI.

```svelte
<script>
  import { report } from './error-reporter.js';
</script>

<svelte:boundary onerror={(e) => report(e)}>
  <FlakyComponent />

  {#snippet failed(error, reset)}
    <button onclick={reset}>oops! try again</button>
  {/snippet}
</svelte:boundary>
```

### Important: what `onerror` does **not** do

- It does **not** prevent the error from being shown via the `failed` snippet.
- If `onerror` itself throws (or you rethrow the error), the error is forwarded to a **parent** boundary if one exists.
- Errors outside the rendering process (event handlers, `setTimeout` callbacks, async work) are **not** caught.

---

## 5. `onerror` Outside the Boundary (using `reset` remotely)

You can capture `error` and `reset` from `onerror` into reactive state, then render a recovery UI anywhere in the tree — even outside the boundary itself.

```svelte
<script>
  let error = $state(null);
  let reset = $state(() => {});

  function onerror(e, r) {
    error = e;
    reset = r;
  }
</script>

<svelte:boundary {onerror}>
  <FlakyComponent />
</svelte:boundary>

{#if error}
  <div class="global-error">
    <p>Remote error UI: {error.message}</p>
    <button onclick={() => {
      error = null;
      reset();
    }}>
      oops! try again
    </button>
  </div>
{/if}
```

This pattern is useful for showing a single, application-wide error toast triggered by any boundary.

---

## 6. Combining `pending` + `failed` + `onerror`

All three can be combined. `onerror` fires alongside the `failed` render, and `pending` only shows on the initial mount.

```svelte
<script>
  async function loadUser(id) {
    await new Promise((r) => setTimeout(r, 1000));
    if (id === 'broken') throw new Error('User not found');
    return { id, name: 'Ada' };
  }

  import { report } from './error-reporter.js';
  let { userId } = $props();
</script>

<svelte:boundary onerror={(e, reset) => report(e, { tag: 'user-card' })}>
  <UserCard user={await loadUser(userId)} />

  {#snippet pending()}
    <div class="skeleton">Loading user…</div>
  {/snippet}

  {#snippet failed(error, reset)}
    <div class="error">
      <p>Failed to load user: {error.message}</p>
      <button onclick={reset}>Retry</button>
    </div>
  {/snippet}
</svelte:boundary>
```

---

## 7. `pending` Does Not Re-Show on Updates

A common point of confusion: once the initial await has resolved, subsequent async updates do **not** re-trigger the `pending` snippet. Use [`$effect.pending()`](https://svelte.dev/docs/svelte/$effect#$effect.pending) to track in-flight async work after the initial render.

```svelte
<script>
  let page = $state(1);
  let data = $derived(await fetchPage(page));

  let inFlight = false;
  $effect(() => {
    $effect.pending(); // tracks in-flight async dependencies
    inFlight = true;
    return () => { inFlight = false; };
  });
</script>

<svelte:boundary>
  {#await fetchPage(page)}
    <p>Initial pending only</p>
  {:then result}
    <List items={result} />
  {/await}

  {#snippet pending()}
    <p>Loading initial data…</p>
  {/snippet}

  {#snippet failed(error, reset)}
    <button onclick={reset}>Try again</button>
  {/snippet}
</svelte:boundary>
```

> Note: in the Svelte playground, your app is rendered inside a boundary with an empty `pending` snippet, so you can use `await` without writing your own.

---

## 8. Nested Boundaries (Granular Error Isolation)

Inner boundaries catch errors first; only errors that bubble out (rethrown from `onerror`, or thrown from inside `failed`) reach the parent.

```svelte
<svelte:boundary>
  <Header />

  <svelte:boundary>
    <Sidebar />
    {#snippet failed(error, reset)}
      <p>Sidebar failed: {error.message}</p>
      <button onclick={reset}>Reload sidebar</button>
    {/snippet}
  </svelte:boundary>

  <svelte:boundary>
    <MainContent />
    {#snippet failed(error, reset)}
      <p>Main content failed: {error.message}</p>
      <button onclick={reset}>Reload main</button>
    {/snippet}
  </svelte:boundary>

  {#snippet failed(error, reset)}
    <p>The whole page crashed: {error.message}</p>
    <button onclick={reset}>Reload page</button>
  {/snippet}
</svelte:boundary>
```

This is the typical "shell with independently-recoverable regions" pattern (e.g. a dashboard with a feed, a chart, and a notifications panel — each can fail and be reset without nuking the whole page).

---

## 9. Forwarding Errors to a Parent Boundary

If you want the inner boundary to render a small fallback but still report the error to the parent, just rethrow inside `onerror`:

```svelte
<script>
  import { report } from './error-reporter.js';
</script>

<svelte:boundary onerror={(e) => { report(e); throw e; }}>
  <UserCard />
  {#snippet failed(error, reset)}
    <p>Couldn't show user card.</p>
    <button onclick={reset}>Retry</button>
  {/snippet}
</svelte:boundary>
```

The inner boundary shows its `failed` UI **and** the parent's `failed` UI fires. The error is also forwarded.

---

## 10. Reset Triggered from Anywhere

Because `reset` is just a function, you can trigger recovery from any UI element — a header banner, a toast, even a keyboard shortcut.

```svelte
<script>
  let boundaryReset = $state(() => {});

  function onerror(error, reset) {
    boundaryReset = reset;
  }

  function handleKeydown(event) {
    if (event.key === 'F5' || (event.key === 'r' && event.metaKey)) {
      event.preventDefault();
      boundaryReset();
    }
  }
</script>

<svelte:window onkeydown={handleKeydown} />

<svelte:boundary {onerror}>
  <FlakyComponent />
  {#snippet failed(error, reset)}
    <p>{error.message}</p>
    <small>Press Cmd/Ctrl+R to retry</small>
  {/snippet}
</svelte:boundary>
```

---

## 11. SSR with `transformError` (Svelte 5.51+)

By default, error boundaries have **no effect on the server** — if an error occurs during server rendering, the whole `render(...)` call fails.

Since 5.51 you can pass `transformError` to `render(...)` (or `mount` / `hydrate`) to control what the `failed` snippet receives on the server. The function must return a **JSON-stringifiable** object, which will be serialized into the HTML and used to hydrate the snippet in the browser.

```js
// server.js
// @errors: 1005
import { render } from 'svelte/server';
import App from './App.svelte';

const { head, body } = await render(App, {
  transformError: (error) => {
    // log the original error, with the stack trace
    console.error(error);

    // return a sanitized user-friendly error
    // (NEVER send raw `message` / `stack` to the browser)
    return {
      message: 'An error occurred!'
    };
  }
});
```

```svelte
<!-- App.svelte -->
<svelte:boundary>
  <FlakyComponent />
  {#snippet failed(error, reset)}
    <p>{error.message}</p>
    <button onclick={reset}>Retry</button>
  {/snippet}
</svelte:boundary>
```

Notes:
- The `failed` snippet is rendered on the server with the **transformed** error object.
- On the client, the snippet hydrates with the deserialized object.
- If the boundary has an `onerror` handler, it is called on hydration with the deserialized error.
- If `transformError` itself throws, the whole `render(...)` fails with that error.
- SvelteKit plans to wire this up automatically via the `handleError` hook; most framework users will not call `render(...)` directly.

---

## 12. `transformError` for `mount` and `hydrate`

The `mount(...)` and `hydrate(...)` functions also accept a `transformError` option (defaulting to the identity function). It transforms a render-time error before it is passed to the `failed` snippet or `onerror` handler.

```js
import { mount, hydrate } from 'svelte';

mount(App, {
  target: document.body,
  transformError: (error) => ({
    message: error.message,
    code: error.code ?? 'UNKNOWN'
  })
});

hydrate(App, {
  target: document.body,
  transformError: (error) => ({ message: 'Hydration failed' })
});
```

---

## 13. `pending` for Skeleton Screens

A common UX pattern: show a skeleton placeholder while the initial data loads, then render the real UI.

```svelte
<script>
  async function fetchDashboard() {
    const res = await fetch('/api/dashboard');
    return res.json();
  }
</script>

<svelte:boundary>
  <Dashboard data={await fetchDashboard()} />

  {#snippet pending()}
    <div class="dashboard-skeleton">
      <div class="skeleton-row" />
      <div class="skeleton-row" />
      <div class="skeleton-row" />
    </div>
  {/snippet}

  {#snippet failed(error, reset)}
    <div class="dashboard-error">
      <p>Dashboard unavailable: {error.message}</p>
      <button onclick={reset}>Retry</button>
    </div>
  {/snippet}
</svelte:boundary>

<style>
  .skeleton-row {
    height: 1.5rem;
    background: linear-gradient(90deg, #eee 25%, #f5f5f5 50%, #eee 75%);
    background-size: 200% 100%;
    animation: shimmer 1.4s infinite;
    border-radius: 4px;
    margin-bottom: 0.5rem;
  }
  @keyframes shimmer {
    0% { background-position: 200% 0; }
    100% { background-position: -200% 0; }
  }
</style>
```

---

## 14. Error Reporting to a Service (Sentry-style)

A realistic example wiring a boundary to a third-party error tracker.

```svelte
<script>
  import * as Sentry from '@sentry/browser';
  import UserCard from './UserCard.svelte';

  function onerror(error, reset) {
    Sentry.captureException(error, {
      tags: { feature: 'user-card' },
      extra: { boundary: 'user-card-boundary' }
    });
  }
</script>

<svelte:boundary {onerror}>
  <UserCard />

  {#snippet failed(error, reset)}
    <div class="error-card">
      <h3>Couldn't load this user</h3>
      <p>We've been notified. Please try again.</p>
      <button onclick={reset}>Retry</button>
    </div>
  {/snippet}
</svelte:boundary>
```

---

## 15. What `<svelte:boundary>` Does NOT Catch

Keep these in mind — they are common sources of confusion:

| Scenario | Caught? |
|----------|---------|
| Error thrown during render | Yes |
| Error in an `$effect` inside the boundary | Yes |
| Rejection of an `await` inside the boundary | Yes |
| Error in a click/input event handler | No |
| Error inside `setTimeout` / `setInterval` | No |
| Error in async work kicked off from an event handler | No |
| Error in a child component *outside* the boundary | No |
| Error thrown from `onerror` or rethrown | Forwarded to parent boundary |

If you need to catch event-handler errors, wrap them manually:

```svelte
<script>
  function handleClick() {
    try {
      mightThrow();
    } catch (e) {
      // rethrow inside a child so the boundary catches it,
      // or use a store / state to surface the error
      reportError(e);
    }
  }
</script>
```

---

## Summary of Patterns

| Pattern | Snippet / Prop | Notes |
|---------|---------------|-------|
| Show spinner while initial data loads | `pending` | Only on initial render |
| Show error fallback with retry | `failed` | Receives `error, reset` |
| Report errors to Sentry/etc. | `onerror` | Fires alongside `failed` |
| Surface error in a global toast | `onerror` + reactive state | `reset` can be stored |
| Recover from a keyboard shortcut | `onerror` + `reset` state | `reset()` is just a function call |
| Isolate independent UI regions | Nested boundaries | Inner fails first; outer catches what bubbles up |
| Render meaningful server errors | `transformError` (5.51+) | Return a JSON-safe object |
| `await` in the playground | Auto-wrapped in empty boundary | No code needed |
