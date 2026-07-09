# Advanced Transition Examples

These examples cover the harder corners of `svelte/transition`: `|global` vs local, deferred crossfades, transition events, and custom CSS/tick transitions.

> See [examples/transition.md](./transition.md) for the basic built-in transition examples.

---

## 1. Local vs Global Transition

```svelte
<script>
  let outer = $state(true);  // outer if
  let inner = $state(true);  // inner if
</script>

<button onclick={() => (outer = !outer)}>toggle outer</button>
<button onclick={() => (inner = !inner)}>toggle inner</button>

{#if outer}
  <div class="outer">
    {#if inner}
      <!-- Local: only animates when inner toggles, not when outer toggles -->
      <p transition:fade>Local fade</p>

      <!-- Global: animates whenever ANY ancestor block toggles -->
      <p transition:fade|global>Global fade</p>
    {/if}
  </div>
{/if}
```

When `outer` flips, only the `|global` element animates. The local one is destroyed instantly because the parent block changed, not the local block.

---

## 2. `onintrostart` / `onintroend` / `onoutrostart` / `onoutroend`

```svelte
<script>
  import { fly } from 'svelte/transition';

  let visible = $state(true);
  let status = $state('idle');
</script>

<button onclick={() => (visible = !visible)}>{visible ? 'hide' : 'show'}</button>
<p>status: {status}</p>

{#if visible}
  <div
    transition:fly={{ y: 200, duration: 800 }}
    onintrostart={() => (status = 'intro started')}
    onintroend={() => (status = 'intro ended')}
    onoutrostart={() => (status = 'outro started')}
    onoutroend={() => (status = 'outro ended')}
  >
    watch my lifecycle
  </div>
{/if}
```

These events bubble like normal DOM events. Order on intro: `introstart` → ... animation ... → `introend`.

---

## 3. Custom Transition Using `css`

`css` is preferred over `tick` because web animations run off the main thread:

```svelte
<script>
  import { elasticOut } from 'svelte/easing';

  function whoosh(node, params) {
    const existing = getComputedStyle(node).transform.replace('none', '');
    return {
      delay: params.delay ?? 0,
      duration: params.duration ?? 400,
      easing: params.easing ?? elasticOut,
      css: (t, u) => `transform: ${existing} scale(${t});`
    };
  }
</script>

{#if visible}
  <div in:whoosh={{ duration: 600 }}>whoosh</div>
{/if}
```

`t` runs from `0` to `1` on intro, `1` to `0` on outro. `u = 1 - t`. Easing is applied to `t` before it reaches `css`.

---

## 4. Custom Transition Using `tick` (When CSS Not Enough)

```svelte
<script>
  function typewriter(node, { speed = 1 }) {
    const valid =
      node.childNodes.length === 1 &&
      node.childNodes[0].nodeType === Node.TEXT_NODE;
    if (!valid) {
      throw new Error('typewriter requires a single text node child');
    }
    const text = node.textContent;
    const duration = text.length / (speed * 0.01);

    return {
      duration,
      tick: (t) => {
        const i = ~~(text.length * t);
        node.textContent = text.slice(0, i);
      }
    };
  }
</script>

{#if visible}
  <p in:typewriter={{ speed: 1 }}>The quick brown fox</p>
{/if}
```

> [!NOTE] If it's possible to use `css` instead of `tick`, do so — web animations can run off the main thread, preventing jank on slower devices.

---

## 5. Deferred Transition (Pair Coordination via Function Return)

If a transition returns a **function** instead of a config object, Svelte calls it in the next microtask. This lets two transitions coordinate — used by `crossfade`:

```svelte
<script>
  import { fade } from 'svelte/transition';

  // A custom transition that defers
  function deferredTransition(node, params) {
    return () => {
      // runs in next microtask after Svelte decides whether to pair it
      return {
        duration: 300,
        css: (t) => `opacity: ${t}`
      };
    };
  }
</script>

{#if show}
  <div in:deferredTransition>fades in (deferred)</div>
{/if}
```

This pattern is what `crossfade` uses internally to send an outgoing element and receive an incoming one at the same position.

---

## 6. `crossfade` — Paired Send/Receive

```svelte
<script>
  import { crossfade } from 'svelte/transition';
  import { quintOut } from 'svelte/easing';

  const [send, receive] = crossfade({
    duration: 400,
    easing: quintOut,
    fallback(node, params) {
      // used when no matching send/receive pair exists
      return {
        duration: 200,
        css: (t) => `opacity: ${t}`
      };
    }
  });

  let items = $state([
    { id: 1, text: 'first' },
    { id: 2, text: 'second' },
    { id: 3, text: 'third' }
  ]);
  let selected = $state(1);
</script>

<div class="row">
  {#each items as item (item.id)}
    <div
      class="item"
      class:selected={selected === item.id}
      onclick={() => (selected = item.id)}
      in:receive={{ key: item.id }}
      out:send={{ key: item.id }}
    >
      {item.text}
    </div>
  {/each}
</div>
```

When `selected` changes (and items re-render), the elements move smoothly via FLIP-style animation rather than fading out/in independently.

---

## 7. `crossfade` Between Lists

```svelte
<script>
  import { crossfade } from 'svelte/transition';

  const [send, receive] = crossfade({ duration: 300 });

  let left = $state(['apple', 'banana']);
  let right = $state(['cherry', 'date']);

  function move(item, from) {
    if (from === 'left') {
      left = left.filter((x) => x !== item);
      right = [...right, item];
    } else {
      right = right.filter((x) => x !== item);
      left = [...left, item];
    }
  }
</script>

<div class="cols">
  <ul>
    {#each left as item (item)}
      <li
        in:receive={{ key: item }}
        out:send={{ key: item }}
        onclick={() => move(item, 'left')}
      >
        {item}
      </li>
    {/each}
  </ul>
  <ul>
    {#each right as item (item)}
      <li
        in:receive={{ key: item }}
        out:send={{ key: item }}
        onclick={() => move(item, 'right')}
      >
        {item}
      </li>
    {/each}
  </ul>
</div>
```

Same `key` (here, `item`) is essential — Svelte matches a `send` to a `receive` with the same key.

---

## 8. Transition With Dynamic Duration

```svelte
<script>
  import { fly } from 'svelte/transition';
  import { quintOut } from 'svelte/easing';

  let visible = $state(false);
  let distance = $state(100);
</script>

<input type="range" min="0" max="500" bind:value={distance} />
<button onclick={() => (visible = !visible)}>toggle</button>

{#if visible}
  <div transition:fly={{ y: distance, duration: distance * 2, easing: quintOut }}>
    flies from {distance}px
  </div>
{/if}
```

When `distance` changes during an active transition, the in-flight transition is updated.

---

## 9. Custom Transition Reading Computed Style

```svelte
<script>
  function rotate(node, { from = 0, to = 360, duration = 600 }) {
    return {
      duration,
      css: (t, u) => `transform: rotate(${from + (to - from) * t}deg); opacity: ${t};`
    };
  }
</script>

{#if visible}
  <div in:rotate={{ from: -90, to: 0, duration: 400 }}>spins in</div>
{/if}
```

`getComputedStyle(node)` lets you preserve existing transforms:

```js
function preserveTransform(node, params) {
  const existing = getComputedStyle(node).transform;
  return {
    duration: params.duration ?? 400,
    css: (t) => `transform: ${existing} translateY(${(1 - t) * 20}px); opacity: ${t};`
  };
}
```

---

## 10. Block-Level Outro Coordination

When a block is outroed, all its descendants outro together — but elements *inside* an outroing block without their own transitions are kept in the DOM until every outro completes:

```svelte
{#if show}
  <div class="card">
    <h1>Title</h1>
    <p transition:slide>Body slides</p>
    <!-- this <img> has no transition but waits for the slide to finish -->
    <img src="/banner.png" alt="" />
  </div>
{/if}
```

The `<img>` is removed from the DOM only after the `<p>` finishes sliding.

---

## 11. Transition That Returns `null` / Disabled

To conditionally skip a transition, return an empty object — but note that the transition still runs and instantly completes:

```svelte
<script>
  import { fade } from 'svelte/transition';

  let visible = $state(true);
  let reducedMotion = $state(false);

  function maybeFade(node) {
    if (reducedMotion) return { duration: 0 };
    return fade(node, { duration: 300 });
  }
</script>

<label><input type="checkbox" bind:checked={reducedMotion} /> reduced motion</label>
<button onclick={() => (visible = !visible)}>toggle</button>

{#if visible}
  <div transition:maybeFade>respects prefers</div>
{/if}
```

For production, gate this with `matchMedia('(prefers-reduced-motion: reduce)')`.

---

## 12. Transition Events on `|global`

```svelte
{#each items as item (item.id)}
  <li
    transition:fly|global={{ y: 20 }}
    onintrostart={() => console.log(item.id, 'enter')}
    onoutroend={() => console.log(item.id, 'leave')}
  >
    {item.text}
  </li>
{/each}
```

Events still fire for `|global` transitions. The `|global` modifier only affects *when* the transition triggers, not the events.

---

## Summary

| Feature | Use Case |
|---------|----------|
| `transition:fade|global` | Animate even when ancestor blocks change |
| `onintrostart/end`, `onoutrostart/end` | Hook into transition lifecycle |
| `css: (t, u) => string` | Preferred — runs off main thread |
| `tick: (t, u) => void` | Use only when CSS can't (e.g. typewriter) |
| Return a function from transition | Deferred coordination (crossfade) |
| `crossfade({ fallback })` | Paired send/receive for moving items |
| `duration: 0` | Disable transition (e.g. reduced-motion) |

See [references/transitions-complete.md](../references/transitions-complete.md) for the full built-in transition API.