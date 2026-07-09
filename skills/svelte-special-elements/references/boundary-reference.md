# svelte:boundary Reference

`<svelte:boundary>` (added in Svelte **5.3.0**) isolates part of the component tree so you can show fallback UI for in-flight `await` expressions and for errors that occur during rendering or in effects.

```svelte
<svelte:boundary onerror={handler}>
  <!-- children -->
  {#snippet pending()}<p>loading…</p>{/snippet}
  {#snippet failed(error, reset)}
    <button onclick={reset}>oops! try again</button>
  {/snippet}
</svelte:boundary>
```

---

## Purpose

Boundaries let you "wall off" parts of your app so that you can:

- provide UI that should be shown when [`await`](https://svelte.dev/docs/svelte/await-expressions) expressions are first resolving, and
- handle errors that occur during rendering or while running effects, and provide UI that should be rendered when an error happens.

If a boundary handles an error (with a `failed` snippet, an `onerror` handler, or both), its existing content is **removed** and replaced with the `failed` snippet output.

> [!NOTE]
> Errors occurring outside the rendering process — for example in event handlers, in `setTimeout` / `setInterval` callbacks, or after async work kicked off from an event — are **not** caught by error boundaries.

---

## Properties

For a boundary to do anything, one or more of the following must be provided.

| Property | Type | Description |
|----------|------|-------------|
| `pending` | `Snippet` | Shown until every `await` inside the boundary has resolved for the first time. |
| `failed` | `Snippet` | Shown when an error is thrown inside the boundary. Receives `(error, reset)`. |
| `onerror` | `(error: unknown, reset: () => void) => void` | Called with the same `error` and `reset`. Useful for error reporting. |

### `pending` snippet

Shown when the boundary is first created. Stays visible until every [`await`](https://svelte.dev/docs/svelte/await-expressions) expression inside the boundary has resolved.

```svelte
<svelte:boundary>
  <p>{await delayed('hello!')}</p>

  {#snippet pending()}
    <p>loading...</p>
  {/snippet}
</svelte:boundary>
```

- Only shown for the **initial** render.
- For subsequent async updates (e.g. the user navigates to a new page that re-triggers an `await`), use [`$effect.pending()`](https://svelte.dev/docs/svelte/$effect#$effect.pending) instead.
- In the [Svelte playground](https://svelte.dev/playground), the whole app is rendered inside a boundary with an empty `pending` snippet, so you can use `await` freely.

### `failed` snippet

Rendered when an error is thrown inside the boundary. Receives `error` and a `reset` function that recreates the contents (clears the error and re-runs the children).

```svelte
<svelte:boundary>
  <FlakyComponent />

  {#snippet failed(error, reset)}
    <button onclick={reset}>oops! try again</button>
  {/snippet}
</svelte:boundary>
```

The `failed` snippet can be passed two ways:

- **Implicitly** — declared as a child snippet of the boundary (as above).
- **Explicitly** — passed as a property, like any other component snippet:

  ```svelte
  <svelte:boundary {failed}>...</svelte:boundary>
  ```

  This mirrors [how snippets are passed to components](https://svelte.dev/docs/svelte/snippet#Passing-snippets-to-components).

### `onerror` handler

Receives the same `error` and `reset` arguments as `failed`. Most useful for forwarding to an error reporting service:

```svelte
<svelte:boundary onerror={(e) => report(e)}>
  ...
</svelte:boundary>
```

Or for capturing `error` and `reset` into reactive state, so the recovery UI can live **outside** the boundary:

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
  <button onclick={() => {
    error = null;
    reset();
  }}>
    oops! try again
  </button>
{/if}
```

**Rethrowing / throwing from `onerror`:** if the handler itself throws, or you rethrow the error, it is forwarded to the nearest parent boundary (if one exists).

---

## Using `transformError` (Svelte 5.51+)

By default, error boundaries have **no effect on the server** — if an error occurs during server rendering, the whole `render(...)` call fails.

Since **5.51** you can control this behaviour for boundaries that have a `failed` snippet, by calling [`render(...)`](https://svelte.dev/docs/svelte/imperative-component-api#render) with a `transformError` function.

```js
// @errors: 1005
import { render } from 'svelte/server';
import App from './App.svelte';

const { head, body } = await render(App, {
  transformError: (error) => {
    // log the original error, with the stack trace
    console.error(error);

    // return a sanitized user-friendly error
    // to display in the `failed` snippet
    return {
      message: 'An error occurred!'
    };
  }
});
```

### Rules for `transformError`

- It must return a **JSON-stringifiable** object. That object is serialized into the HTML, then deserialized on the client and used to render the `failed` snippet during hydration.
- If `transformError` throws (or rethrows), the whole `render(...)` call fails with that error.
- If the boundary has an `onerror` handler, it is called on hydration with the deserialized error object.
- `mount(...)` and `hydrate(...)` also accept a `transformError` option (defaults to the identity function). It transforms a render-time error before it is passed to a `failed` snippet or `onerror` handler.

> [!NOTE]
> SvelteKit users typically do not have direct access to the `render(...)` call — the framework must configure `transformError` on your behalf. SvelteKit will wire this up via the [`handleError`](https://svelte.dev/docs/kit/hooks#Shared-hooks-handleError) hook.

> [!WARNING]
> Errors thrown during server-side rendering can contain sensitive information in the `message` and `stack`. Redact these rather than sending them unaltered to the browser.

### `transformError` for `mount` / `hydrate`

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

## What `<svelte:boundary>` Catches

| Scenario | Caught? | Notes |
|----------|---------|-------|
| Error thrown during render of a child | Yes | Replaced by `failed` |
| Rejection of `await` inside the boundary | Yes | `pending` may be shown first if it is the initial render |
| Error in an `$effect` inside the boundary | Yes | |
| `{#await ...}` block inside the boundary | Yes | |
| Error in a click / input event handler | No | Wrap manually with `try/catch` |
| Error inside `setTimeout` / `setInterval` | No | |
| Async work kicked off from an event handler | No | |
| Error in a sibling **outside** the boundary | No | Wrap that sibling in its own boundary |
| Error thrown from `onerror` (or rethrown) | Forwarded | Handled by the nearest parent boundary |

---

## Nesting

Inner boundaries catch errors first. An error that escapes an inner boundary (e.g. thrown from `onerror`, or simply not handled) bubbles to the nearest parent.

```svelte
<svelte:boundary>
  <Header />

  <svelte:boundary>
    <Sidebar />
    {#snippet failed(error, reset)}
      <button onclick={reset}>Reload sidebar</button>
    {/snippet}
  </svelte:boundary>

  <svelte:boundary>
    <MainContent />
    {#snippet failed(error, reset)}
      <button onclick={reset}>Reload main</button>
    {/snippet}
  </svelte:boundary>

  {#snippet failed(error, reset)}
    <button onclick={reset}>Reload page</button>
  {/snippet}
</svelte:boundary>
```

A common pattern is a "shell" boundary for catastrophic failures, with per-region boundaries inside for granular recovery.

---

## When to Use

| Use case | Boundary shape |
|----------|----------------|
| Show a spinner / skeleton for initial data load | `pending` |
| Render an error fallback with a retry button | `failed` |
| Report errors to Sentry / Bugsnag / etc. | `onerror` |
| Surface errors in a global toast | `onerror` + reactive state |
| Recover with a keyboard shortcut | `onerror` + `reset` state |
| Isolate independent UI regions | Nested boundaries |
| Render meaningful errors on the server | `transformError` (5.51+) |

---

## Common Pitfalls

1. **`pending` does not re-show after the initial mount.** Use `$effect.pending()` for subsequent async work.
2. **Event-handler errors are not caught.** Wrap them in `try/catch` and rethrow into a child, or surface them via state.
3. **`failed` is the only thing rendered after an error.** If you also want the original content to reappear, call `reset()`.
4. **The playground auto-wraps your app in a boundary.** Don't be surprised that bare `await` works there.
5. **Throwing from `onerror` bubbles up.** If you want a parent boundary to handle the error *and* the current boundary to render its own `failed` snippet, throw inside `onerror`.
6. **`transformError` must return JSON.** Don't return `Error` instances, functions, or `Map`/`Set` — they will be lost on serialization.

---

## API Summary

```ts
type Snippet<T extends unknown[] = []> = (...args: T) => string;

interface BoundaryProps {
  /** Snippet shown until the first batch of `await`s resolves. */
  pending?: Snippet;

  /** Snippet shown when an error is thrown. Receives (error, reset). */
  failed?: Snippet<[error: unknown, reset: () => void]>;

  /** Called with (error, reset) when an error is thrown. */
  onerror?: (error: unknown, reset: () => void) => void;
}

interface RenderOptions {
  /** Transform a render-time error before passing it to `failed` / `onerror`. */
  transformError?: (error: unknown) => Record<string, unknown>;
}
```
