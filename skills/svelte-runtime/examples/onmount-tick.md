# Lifecycle Hooks: onMount, onDestroy, tick

Practical examples for the Svelte 5 lifecycle API (`onMount`, `onDestroy`, `tick`) and how they interact with the imperative `mount`/`unmount`/`hydrate` runtime.

> Svelte 5 only has two lifecycle phases — **creation** and **destruction**. There are no "before update"/"after update" hooks because the smallest unit of change is the (render) effect, not the component. Use `$effect`, `$effect.pre`, and `tick` for granular reactivity.

---

## 1. Basic onMount with Cleanup

A classic interval example. The function returned from `onMount` is called on unmount.

```svelte
<!-- Clock.svelte -->
<script>
  import { onMount } from 'svelte';

  let time = $state(new Date());

  onMount(() => {
    const id = setInterval(() => {
      time = new Date();
    }, 1000);

    // Cleanup function — runs on unmount
    return () => clearInterval(id);
  });
</script>

<p>Current time: {time.toLocaleTimeString()}</p>
```

```js
// main.js
import { mount, unmount } from 'svelte';
import Clock from './Clock.svelte';

const app = mount(Clock, { target: document.body });

// Later...
unmount(app); // clearInterval fires here
```

> The cleanup function **only runs when the callback passed to `onMount` is synchronous**. Async functions always return a `Promise`, which Svelte treats as the return value.

---

## 2. onMount Does Not Run During SSR

`onMount` is client-only. Use it freely for things that touch `window`, `document`, or `IntersectionObserver` — no need for `if (browser)` guards.

```svelte
<!-- ViewportTracker.svelte -->
<script>
  import { onMount } from 'svelte';

  let visible = $state(false);
  let el = $state();

  onMount(() => {
    const observer = new IntersectionObserver(([entry]) => {
      visible = entry.isIntersecting;
    });
    observer.observe(el);
    return () => observer.disconnect();
  });
</script>

<div bind:this={el}>
  {visible ? 'in viewport' : 'offscreen'}
</div>
```

**SSR behaviour:** the component renders fine on the server, the `onMount` simply never fires there. `visible` defaults to `false` on the server, then updates after hydration.

---

## 3. onMount with Async Work + AbortController

When you need to do async work in `onMount`, you cannot rely on the returned cleanup (because async returns a Promise). Instead, use `AbortController` or a `cancelled` flag.

```svelte
<!-- UserLoader.svelte -->
<script>
  import { onMount } from 'svelte';

  let user = $state(null);
  let error = $state(null);

  let { userId } = $props();

  onMount(() => {
    const controller = new AbortController();

    (async () => {
      try {
        const res = await fetch(`/api/users/${userId}`, {
          signal: controller.signal
        });
        user = await res.json();
      } catch (e) {
        if (e.name !== 'AbortError') error = e;
      }
    })();

    return () => controller.abort();
  });
</script>

{#if error}
  <p class="error">{error.message}</p>
{:else if user}
  <h1>{user.name}</h1>
{:else}
  <p>Loading…</p>
{/if}
```

**Why the IIFE?** The outer arrow must return the cleanup synchronously. Wrap the async body in an IIFE so the outer `return () => controller.abort()` is what `onMount` actually receives.

---

## 4. onDestroy Runs on Both Server and Client

`onDestroy` is the only lifecycle hook that runs in SSR components. Use it for resource cleanup that must happen regardless of environment.

```svelte
<!-- ResourceHolder.svelte -->
<script>
  import { onDestroy } from 'svelte';

  // Pretend this opens a DB connection
  const connection = openConnection();

  onDestroy(() => {
    connection.close();
  });
</script>

<p>Connected to {connection.name}</p>
```

```js
// Server
import { render } from 'svelte/server';
const { body } = await render(ResourceHolder);
// onDestroy fires after render completes — connection is closed
```

---

## 5. tick() to Focus an Element After Render

`tick()` resolves once all pending state changes have been applied to the DOM. The canonical use is focusing an input after a state-driven mount.

```svelte
<!-- SearchBox.svelte -->
<script>
  import { tick } from 'svelte';

  let open = $state(false);
  let input = $state();

  async function show() {
    open = true;
    await tick();   // wait for the <input> to be in the DOM
    input?.focus(); // safe — element now exists
  }
</script>

<button onclick={show}>Search</button>

{#if open}
  <input bind:this={input} placeholder="Type to search…" />
{/if}
```

Without `await tick()`, the conditional block hasn't been rendered yet, so `input` is `undefined`.

---

## 6. tick() with $effect.pre to Measure Before Update

Pair `$effect.pre` with `tick()` to implement the chat autoscroll pattern. The example below scrolls to the bottom only when the user was already pinned to the bottom.

```svelte
<!-- Chat.svelte -->
<script>
  import { tick } from 'svelte';

  let messages = $state([]);
  let theme = $state('dark');
  let viewport = $state();

  $effect.pre(() => {
    // Read `messages` so the effect re-runs only when messages change
    messages;
    if (!viewport) return;

    const wasPinned =
      viewport.scrollHeight - viewport.scrollTop - viewport.clientHeight < 50;

    if (wasPinned) {
      tick().then(() => {
        viewport.scrollTop = viewport.scrollHeight;
      });
    }
  });

  function send(text) {
    messages = [...messages, text];
  }
</script>

<div class:dark={theme === 'dark'}>
  <div bind:this={viewport}>
    {#each messages as m}
      <p>{m}</p>
    {/each}
  </div>
  <button onclick={() => theme = theme === 'dark' ? 'light' : 'dark'}>
    toggle theme
  </button>
</div>
```

`$effect.pre` runs **before** the DOM update, so we can measure the old scroll position. `tick()` then resolves **after** the DOM has been updated with the new message, at which point we set `scrollTop`.

---

## 7. tick() vs flushSync()

- `await tick()` — promise-based; lets the rest of your async code run; suitable for most user code.
- `flushSync()` — synchronous; forces all pending effects to run **now**. Use it in tests or when an external library needs DOM to be settled immediately.

```js
import { tick, flushSync } from 'svelte';

// Async-friendly
async function addItem(item) {
  list.push(item);
  await tick();
  scrollToBottom();
}

// Sync (test code, D3 binding, etc.)
function updateAndMeasure() {
  count = count + 1;
  flushSync();
  chart.update(width());
}
```

---

## 8. onMount + flushSync in Tests

By default, `mount()` does not run `$effect` callbacks (they wait for the microtask queue). In tests, use `flushSync()` immediately after `mount`.

```js
import { mount, flushSync, unmount } from 'svelte';
import { test, expect } from 'vitest';
import Header from './Header.svelte';

test('sets the document title on mount', () => {
  const app = mount(Header, {
    target: document.body,
    props: { title: 'Dashboard' }
  });

  // Without flushSync, the effect inside Header has not run yet
  flushSync();

  expect(document.title).toBe('Dashboard');

  unmount(app);
});
```

This applies to `hydrate()` as well.

---

## 9. onMount Calling External Functions

`onMount` may be called from an external module, not just inside a component. This is useful for testing or for shared setup helpers.

```js
// setup.js
import { onMount } from 'svelte';

export function trackPageView(name) {
  onMount(() => {
    analytics.send('pageview', { name });
    return () => analytics.send('pageview-end', { name });
  });
}
```

```svelte
<!-- Page.svelte -->
<script>
  import { trackPageView } from './setup.js';
  trackPageView('home');
</script>

<h1>Home</h1>
```

The function must still be called during component initialisation — but it doesn't have to *live* in the component.

---

## 10. Multiple onMount Calls

You can call `onMount` more than once. Each callback runs (and its cleanup is registered) in order.

```svelte
<script>
  import { onMount } from 'svelte';

  onMount(() => {
    const id = setInterval(tickClock, 1000);
    return () => clearInterval(id);
  });

  onMount(() => {
    document.addEventListener('visibilitychange', onVisChange);
    return () => document.removeEventListener('visibilitychange', onVisChange);
  });
</script>
```

Cleanup order is **LIFO** — the last registered cleanup runs first when the component unmounts.

---

## 11. Lifecycle Phases: Server vs Client

| Phase / Hook | Server (SSR) | Client |
|---|---|---|
| Component constructor | runs | runs |
| `$state` / `$derived` initialisation | runs | runs |
| `onMount` | **skipped** | runs after mount |
| `onDestroy` | runs (after render) | runs before unmount |
| `$effect` | **skipped** | runs after mount |
| `$effect.pre` | **skipped** | runs before each update |
| `tick()` | no-op | waits for next DOM update |
| `flushSync()` | no-op | forces all pending effects to run |

> Effects and `onMount` rely on a microtask, which never fires during synchronous SSR. Use `hydratable()` instead if you need shared data.

---

## 12. onMount vs $effect

Reach for `$effect` when you need to **react to changes**. Reach for `onMount` when you need a **one-shot side effect after the component is in the DOM**.

```svelte
<script>
  import { onMount } from 'svelte';

  let query = $state('');
  let results = $state([]);

  // One-shot: set up the WebSocket once
  onMount(() => {
    const ws = new WebSocket('/search');
    ws.onmessage = (e) => { results = JSON.parse(e.data); };
    return () => ws.close();
  });

  // Reactive: refetch when query changes
  $effect(() => {
    if (!query) return;
    fetch(`/api/search?q=${query}`).then(r => r.json()).then(d => results = d);
  });
</script>
```

---

## 13. Manual mount + onMount Pattern

When you `mount` a component imperatively, `onMount` inside it still fires (after the next microtask). Use this to inject components outside the normal SvelteKit routing tree.

```js
import { mount, unmount } from 'svelte';
import Notification from './Notification.svelte';

let active = null;

export function notify(message) {
  active = mount(Notification, {
    target: document.body,
    props: { message }
  });
  // Notification.svelte calls onMount to start a fade-out timer.
}

export function dismiss() {
  if (active) {
    unmount(active, { outro: true });
    active = null;
  }
}
```

```svelte
<!-- Notification.svelte -->
<script>
  import { onMount } from 'svelte';
  let { message, duration = 3000 } = $props();

  onMount(() => {
    const id = setTimeout(() => dispatch('done'), duration);
    return () => clearTimeout(id);
  });
</script>

{#if message}
  <div class="toast" transition:fade>{message}</div>
{/if}
```

---

## 14. onDestroy with Returns from Synchronous Setup

`onDestroy` runs synchronously and may itself return a cleanup function (in newer Svelte versions) for symmetry, but the more common pattern is just to put cleanup logic in the body.

```svelte
<script>
  import { onMount, onDestroy } from 'svelte';

  let cursor = null;

  onMount(() => {
    cursor = new Locator();
    return () => cursor.destroy();
  });

  // Equivalent alternative
  onDestroy(() => {
    cursor?.destroy();
  });
</script>
```

Prefer `onMount`'s return-value pattern when the resource is created in `onMount`. Use `onDestroy` for resources that come from props, from `<script context="module">`, or from constructor-time code.

---

## 15. Lifecycle During Hydration

`hydrate()` reuses SSR HTML, then re-runs the component's `<script>` to wire up reactivity. Effects and `onMount` then fire on the microtask, exactly like `mount()`.

```js
// Client bootstrap
import { hydrate, flushSync } from 'svelte';
import App from './App.svelte';

// HTML already exists in #app (from SSR)
const app = hydrate(App, {
  target: document.querySelector('#app'),
  props: window.__INITIAL_DATA__
});

// If you have an effect that must run before the first paint logic completes:
flushSync();
```

If a component calls `Math.random()` in its `<script>`, you'll get a **hydration mismatch**. Wrap the random call in `hydratable('seed', () => Math.random())` so the value is serialised on the server and replayed on the client.

---

## Quick Reference

| Hook | Returns cleanup? | SSR? | When to use |
|---|---|---|---|
| `onMount(fn)` | yes (sync only) | no | One-shot after DOM is ready |
| `onDestroy(fn)` | yes | yes | Cleanup that must always run |
| `await tick()` | n/a | no | Wait for next DOM update |
| `$effect(fn)` | yes | no | React to state changes |
| `$effect.pre(fn)` | yes | no | Measure DOM before update |
| `flushSync()` | n/a | no | Force effects to run synchronously |