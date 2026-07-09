# Lifecycle in the Mount / Hydrate Runtime

This reference describes how Svelte 5's lifecycle hooks (`onMount`, `onDestroy`, `tick`, `$effect`, `$effect.pre`) interact with the imperative `mount`, `unmount`, `hydrate`, and `render` APIs.

> **Mental model**: in Svelte 5 the smallest reactive unit is the **effect**, not the component. There is no "before update"/"after update" hook. Instead, `$effect` re-runs only when the state it touches changes, and `$effect.pre` re-runs before the DOM is updated for that change.

---

## 1. The Two Phases

Svelte 5 collapses the lifecycle to two phases:

1. **Creation** — the component script runs, `$state`/`$derived` are initialised, props are bound, and the template is rendered into DOM (either as a string on the server or as nodes on the client).
2. **Destruction** — the component is being torn down; `onDestroy` runs, all effect cleanups run, and the DOM nodes are removed.

Anything "in between" is handled by **effects**, which are scheduled per dependency and have nothing to do with the component as a whole.

---

## 2. What Fires When

### 2.1 During `mount(Component, options)`

| Step | When | Detail |
|---|---|---|
| 1 | synchronous | Component script runs; `$state`/`$derived` initialised |
| 2 | synchronous | Props applied |
| 3 | synchronous | Template rendered into a DocumentFragment |
| 4 | synchronous | Fragment inserted into `target` |
| 5 | synchronous | Instance returned to caller |
| 6 | **microtask** | `$effect` callbacks run |
| 7 | **microtask** | `onMount` callbacks run (after effects) |
| 8 | later | DOM updates flushed; `tick()` resolves |

> **Important**: a freshly-mounted component is **interactive but idle** — its `onMount` and `$effect` callbacks have not fired yet by the time `mount()` returns. If you need them to have run, call `flushSync()` immediately after `mount()` (or `await tick()` for async code).

### 2.2 During `hydrate(Component, options)`

Identical to `mount`, except step 3–4 reuses the existing SSR DOM instead of replacing `target`. Hydration **also** does not run effects synchronously — use `flushSync()` for the same reason.

```js
import { hydrate, flushSync } from 'svelte';
import App from './App.svelte';

const app = hydrate(App, { target: document.querySelector('#app') });
flushSync(); // run pending effects before continuing
```

### 2.3 During `render(Component, options)` (server)

| Step | Detail |
|---|---|
| 1 | Component script runs; `$state`/`$derived` initialised |
| 2 | Props applied |
| 3 | Template rendered to string |
| 4 | `onDestroy` registered and **invoked after render completes** (the only lifecycle that runs on the server) |
| 5 | `{ head, body, hashes? }` returned |

There is no microtask queue on the server; `flushSync()` and `tick()` are no-ops there. `onMount` and `$effect` never run.

### 2.4 During `unmount(app, { outro? })`

| Step | When | Detail |
|---|---|---|
| 1 | synchronous | Effect cleanups run (LIFO) |
| 2 | synchronous | `onDestroy` callbacks run |
| 3 | if `outro: true` | Transitions play; waits for `transitionend`/`animationend` (or 1000 ms timeout) |
| 4 | end | DOM removed; `onMount` cleanup **does not** run here — it ran in step 1 along with all other effect cleanups |

If `outro: false` (default), unmount is synchronous.

---

## 3. `onMount`

### Signature

```ts
function onMount(fn: () => void | (() => void)): void;
```

### Rules

- Must be called during **component initialisation** (the `<script>` block, a function called from it, or a `<script context="module">`-time call site). Calling it later is a no-op.
- Does **not** run during SSR.
- The callback runs on the **next microtask** after mount/hydrate, in registration order.
- If the callback returns a function, it is registered as a cleanup. **The callback must be synchronous** to honour this — `async () => {...}` always returns a `Promise`, which Svelte treats as the return value, not as a cleanup.
- Multiple `onMount` calls are allowed; cleanups run **LIFO**.

### Synchronous cleanup pattern for async work

```svelte
<script>
  import { onMount } from 'svelte';

  onMount(() => {
    const controller = new AbortController();

    (async () => {
      const res = await fetch('/api/data', { signal: controller.signal });
      data = await res.json();
    })();

    // Cleanup returned synchronously
    return () => controller.abort();
  });
</script>
```

---

## 4. `onDestroy`

### Signature

```ts
function onDestroy(fn: () => void): void;
```

### Rules

- Runs synchronously before unmount on the client, and after the server render completes.
- Runs **before** effect cleanups in some Svelte versions; in Svelte 5 the order is: effect cleanups → `onDestroy` callbacks → DOM removal.
- No return-value contract — cleanup logic lives in the body.

```svelte
<script>
  import { onDestroy } from 'svelte';

  const socket = new WebSocket('/ws');
  onDestroy(() => socket.close());
</script>
```

---

## 5. `tick`

### Signature

```ts
function tick(): Promise<void>;
```

### Semantics

- Resolves **once any pending state changes have been applied to the DOM**.
- If there are no pending changes, resolves in the **next microtask** (so it never resolves synchronously).
- Does not run effects; it only waits for them to settle.
- On the server, `tick()` resolves in the next microtask but does no work (there is no DOM and no reactive scheduler).

### Common uses

- **Focus management** after a conditional block opens.
- **Measurements** (`scrollHeight`, `getBoundingClientRect`) after a state change.
- **Scrolling** (chat-style autoscroll) after content is added.

### `tick()` vs `flushSync()`

| Aspect | `tick()` | `flushSync()` |
|---|---|---|
| Returns | Promise | void |
| Style | async/await | sync |
| Effect on scheduler | Waits for next batch | Runs scheduler immediately |
| Side effects | None (read-only) | Forces all pending effects to run |
| Use case | User code, async flows | Tests, external library sync |

### `tick()` vs `Promise.resolve()`

`Promise.resolve()` is one microtask earlier than `tick()` and **does not** wait for Svelte's scheduler. Always use `tick()` when the next thing you do depends on the DOM being up to date.

---

## 6. `$effect` and `$effect.pre`

### Signatures

```ts
$effect(() => { /* side effect */ return () => { /* cleanup */ }; });
$effect.pre(() => { /* runs before DOM update */ });
```

### When they run

- After mount/hydrate, on the first microtask.
- Whenever any reactive value read in the body changes.
- `$effect.pre` runs **before** the DOM update; `$effect` runs **after**.
- They do **not** run during SSR.

### Why `$effect.pre` instead of `beforeUpdate`

`beforeUpdate` ran on **every** component update (e.g. when `theme` changed in the chat example). `$effect.pre` only re-runs when the values it actually reads change — so referencing `messages` but not `theme` gives you surgical dependency tracking.

```svelte
<script>
  import { tick } from 'svelte';
  let messages = $state([]);
  let theme = $state('dark');
  let viewport;

  $effect.pre(() => {
    messages; // <-- explicit dependency
    const wasPinned = viewport && viewport.scrollTop > viewport.scrollHeight - 50;
    if (wasPinned) {
      tick().then(() => { viewport.scrollTop = viewport.scrollHeight; });
    }
  });
</script>
```

---

## 7. Lifecycle Phase Matrix

| API / Hook | Mounts? | Runs effects? | Fires `onMount`? | Runs on server? |
|---|---|---|---|---|
| `mount()` | yes (replaces target) | async (microtask) | async | no |
| `hydrate()` | yes (reuses DOM) | async (microtask) | async | no |
| `render()` (server) | n/a | no | no | yes |
| `unmount()` | destroys | runs cleanups | runs `onMount` cleanup | no (server never mounted it) |
| `onMount(fn)` | n/a | — | schedules `fn` | skipped |
| `onDestroy(fn)` | n/a | — | registers `fn` | runs |
| `$effect` | n/a | registers | registers | skipped |
| `$effect.pre` | n/a | registers (pre-DOM) | registers | skipped |
| `await tick()` | n/a | waits for next batch | — | no-op-ish |
| `flushSync()` | n/a | runs now | runs now | no-op |

---

## 8. Effects and Microtask Scheduling

Effects are scheduled on the **microtask queue**. This means:

1. Multiple state changes in the same synchronous block produce **one** DOM update.
2. `mount()` returns before effects run — by design, so initialisation is cheap.
3. `Promise.resolve().then(...)` runs **before** Svelte's microtask, because Svelte uses `queueMicrotask` (or its equivalent), which is queued after resolved-Promise microtasks.
4. `await tick()` is therefore the correct primitive for "wait until the DOM reflects my latest state changes."

```js
import { tick } from 'svelte';

list.push(newItem);          // mutates $state proxy
await tick();                 // waits one scheduler tick
container.scrollTop = container.scrollHeight; // safe
```

---

## 9. Hydration and Lifecycle

When `hydrate()` runs:

1. The component script **re-executes** on the client. Any `$state(initial)` reads the same `initial`, so reactive state re-initialises identically.
2. The SSR HTML is preserved; Svelte walks it and attaches event listeners.
3. Effects and `onMount` register on the next microtask.
4. `hydratable(key, fn)` short-circuits — instead of running `fn`, it returns the value that was serialised on the server.

For values that are random or time-dependent, **always** wrap them in `hydratable()`:

```ts
import { hydratable } from 'svelte';
const rand = hydratable('random', () => Math.random());
```

If you do not, the SSR HTML and CSR initial render diverge and you get a hydration mismatch.

---

## 10. Tooling Recap

| You want to… | Use |
|---|---|
| Run code once after the component is in the DOM | `onMount(fn)` |
| React to state changes (general side effects) | `$effect(fn)` |
| Measure DOM, then mutate it after the change | `$effect.pre(fn)` + `await tick()` |
| Run cleanup regardless of mount/hydrate | `onDestroy(fn)` or a returned cleanup |
| Force effects to run synchronously (tests) | `flushSync()` |
| Wait for the next DOM update (user code) | `await tick()` |
| Run code on the server only | `onDestroy` or `if (server)` |

---

## 11. Common Pitfalls

1. **Async `onMount` returns a Promise.** The cleanup you intended is silently discarded. Wrap async bodies in an IIFE so the outer arrow returns the cleanup synchronously.
2. **`mount` doesn't run effects.** If your test asserts a DOM side-effect that lives in `$effect`, call `flushSync()` after `mount()`.
3. **`hydrate` re-executes the script.** Server-only assumptions (`window`, `crypto.randomUUID()` without `hydratable`) will produce a hydration mismatch.
4. **`tick()` doesn't run effects**, it only waits for them. To **force** them, use `flushSync()`.
5. **Cleanup order is LIFO** for multiple `onMount` calls. Don't rely on a specific order unless you register them in a specific order.
6. **`onDestroy` runs on the server too** — keep it free of `window`/`document` references.

---

## 12. Quick Decision Tree

```
Need to run code after first paint?
├── Yes, reactively on state changes → $effect
├── Yes, but only once after mount → onMount
└── Need synchronous guarantee → flushSync after mount

Need to wait for DOM to settle?
├── Yes, async/await friendly → await tick()
└── Yes, blocking → flushSync()

Need to clean up?
└── Return cleanup from onMount, or register onDestroy
```