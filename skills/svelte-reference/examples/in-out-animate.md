# `in:` / `out:` / `animate:` Examples

Three closely-related directives that are easy to conflate:

| Directive | When it runs | Reversible? |
|-----------|--------------|-------------|
| `transition:fade` | entering *or* leaving | yes (reverses mid-flight) |
| `in:fade` | entering only | no |
| `out:fade` | leaving only | no |
| `animate:flip` | keyed each block reorders | n/a (movement) |

---

## 1. `in:` + `out:` — Different Enter vs Exit

```svelte
<script>
  import { fade, fly } from 'svelte/transition';

  let visible = $state(false);
</script>

<label><input type="checkbox" bind:checked={visible} /> visible</label>

{#if visible}
  <div in:fly={{ y: 200 }} out:fade>flies in, fades out</div>
{/if}
```

Unlike `transition:fly` (which would *reverse* the fly when toggled off), `in:fly` plays out fully, then `out:fade` plays. If you toggle off mid-intro, the in-transition is **abandoned** and the out-transition restarts from t=0.

---

## 2. `in:` and `out:` With the Same Transition, Different Params

```svelte
<script>
  import { fade } from 'svelte/transition';
</script>

{#if visible}
  <!-- fade in immediately, but linger on the way out -->
  <div in:fade={{ delay: 0 }} out:fade={{ delay: 200, duration: 800 }}>
    staggered
  </div>
{/if}
```

A common "polite dismissal" pattern.

---

## 3. Asymmetric `transition:` vs `in:` + `out:`

```svelte
{#if x}
  <!-- bidirectional: reverses on toggle -->
  <div transition:fade>fade both ways</div>

  <!-- asymmetric: fly in, fade out -->
  <div in:fly={{ y: 50 }} out:fade>asymmetric</div>

  <!-- staggered: same transition, out delayed -->
  <div transition:fade={{ duration: 300, delay: x ? 0 : 500 }}>
    staggered
  </div>
{/if}
```

---

## 4. `animate:flip` — Basic List Reordering

```svelte
<script>
  import { flip } from 'svelte/animate';
  import { quintOut } from 'svelte/easing';

  let items = $state(['apple', 'banana', 'cherry', 'date']);
</script>

<button onclick={() => items = items.slice().reverse()}>reverse</button>
<button onclick={() => items = items.slice().sort()}>sort</button>

<ul>
  {#each items as item (item)}
    <li animate:flip={{ duration: 400, easing: quintOut }}>
      {item}
    </li>
  {/each}
</ul>
```

FLIP = First, Last, Invert, Play. Svelte measures positions before and after, then animates the delta with a `transform`.

**The key `(item)` is critical.** Without a stable key, Svelte can't know which element moved.

---

## 5. `animate:flip` With Dynamic Duration

```svelte
<script>
  import { flip } from 'svelte/animate';
  let items = $state([1, 2, 3, 4, 5]);
</script>

<ul>
  {#each items as item (item)}
    <li animate:flip={{ duration: 300 }}>
      {item}
    </li>
  {/each}
</ul>

<button onclick={() => (items = items.slice().reverse())}>reverse</button>
```

You can also pass `duration` as a function — see [references/animations-reference.md](../references/animations-reference.md).

---

## 6. Custom Animation Function (`animate:whizz`)

```svelte
<script>
  import { cubicOut } from 'svelte/easing';

  /**
   * @param {HTMLElement} node
   * @param {{ from: DOMRect, to: DOMRect }} states
   * @param {any} _params
   */
  function whizz(node, { from, to }, _params) {
    const dx = from.left - to.left;
    const dy = from.top - to.top;
    const distance = Math.sqrt(dx * dx + dy * dy);

    return {
      delay: 0,
      duration: Math.sqrt(distance) * 120,
      easing: cubicOut,
      css: (t, u) =>
        `transform: translate(${u * dx}px, ${u * dy}px) rotate(${t * 360}deg);`
    };
  }

  let items = $state(['A', 'B', 'C']);
</script>

<button onclick={() => (items = items.slice().reverse())}>reverse</button>

<ul>
  {#each items as item (item)}
    <li animate:whizz>{item}</li>
  {/each}
</ul>
```

The `from`/`to` rectangles are the element's bounding boxes before and after the reorder. Use the delta to compute distance, angle, etc.

---

## 7. Custom Animation With `tick` (Imperative)

When CSS can't express what you need (e.g. animating color *and* transform together, or text content):

```svelte
<script>
  import { cubicOut } from 'svelte/easing';

  /**
   * @param {HTMLElement} node
   * @param {{ from: DOMRect, to: DOMRect }} _states
   */
  function colorFlip(node, _states) {
    return {
      duration: 600,
      easing: cubicOut,
      tick: (t) => {
        node.style.color = t > 0.5 ? 'Pink' : 'Cyan';
        node.style.transform = `scale(${0.8 + 0.2 * t})`;
      }
    };
  }

  let items = $state(['one', 'two', 'three']);
</script>

<button onclick={() => (items = items.slice().reverse())}>reverse</button>

<ul>
  {#each items as item (item)}
    <li animate:colorFlip>{item}</li>
  {/each}
</ul>
```

> [!NOTE] If it's possible to use `css` instead of `tick`, do so — web animations can run off the main thread, preventing jank on slower devices.

---

## 8. Animation Parameters Passed Through

```svelte
<script>
  import { flip } from 'svelte/animate';

  /**
   * @param {HTMLElement} node
   * @param {{ from: DOMRect, to: DOMRect }} _states
   * @param {{ baseDuration: number }} params
   */
  function whizz(node, { from, to }, { baseDuration = 300 }) {
    const dx = from.left - to.left;
    const dy = from.top - to.top;
    const distance = Math.sqrt(dx * dx + dy * dy);

    return {
      duration: baseDuration + distance,
      css: (t, u) => `transform: translate(${u * dx}px, ${u * dy}px);`
    };
  }

  let items = $state(['A', 'B', 'C', 'D']);
</script>

<button onclick={() => (items = items.slice().sort((a, b) => b.localeCompare(a)))}>reverse sort</button>

<ul>
  {#each items as item (item)}
    <li animate:whizz={{ baseDuration: 200 }}>{item}</li>
  {/each}
</ul>
```

---

## 9. `animate:` Only Runs on Reorder — Not Add/Remove

```svelte
<script>
  import { flip } from 'svelte/animate';

  let items = $state(['A', 'B', 'C']);

  function add() { items = [...items, 'D']; }
  function remove() { items = items.slice(0, -1); }
  function shuffle() { items = items.slice().sort(() => Math.random() - 0.5); }
</script>

<button onclick={add}>add (no animate)</button>
<button onclick={remove}>remove (no animate)</button>
<button onclick={shuffle}>shuffle (animate:flip runs)</button>

<ul>
  {#each items as item (item)}
    <li animate:flip={{ duration: 400 }}>{item}</li>
  {/each}
</ul>
```

Adding/removing doesn't trigger `animate:` — it only runs when the *index of an existing item changes*.

---

## 10. `animate:` Must Be on an Immediate Child of `{#each ... (key)}`

```svelte
<!-- WRONG: animate: on a non-direct child -->
{#each items as item (item)}
  <div>
    <span animate:flip>{item}</span>  <!-- does NOT animate -->
  </div>
{/each}

<!-- RIGHT: animate: on the direct child of the keyed each -->
{#each items as item (item)}
  <div animate:flip>
    <span>{item}</span>  <!-- moves with the div -->
  </div>
{/each}
```

`animate:flip` measures the element it's attached to. Move it up the tree if you want a larger container to slide.

---

## 11. Combining `animate:flip` With `transition:` for List Mutations

```svelte
<script>
  import { flip } from 'svelte/animate';
  import { fade } from 'svelte/transition';

  let items = $state(['A', 'B', 'C']);
</script>

<ul>
  {#each items as item (item)}
    <li animate:flip={{ duration: 300 }} in:fade out:fade>
      {item}
    </li>
  {/each}
</ul>
```

`in:fade` / `out:fade` play when items are added or removed. `animate:flip` plays when items reorder. They compose cleanly.

---

## 12. `in:` / `out:` With Shared Configuration

```svelte
<script>
  import { fly, fade } from 'svelte/transition';
  import { cubicOut } from 'svelte/easing';

  const flyConfig = { y: 50, duration: 400, easing: cubicOut };
</script>

{#if visible}
  <!-- spread the config -->
  <div in:fly={flyConfig} out:fade={{ duration: 200 }}>
    configurable
  </div>
{/if}
```

The double `{{curlies}}` in `transition:fade={{ duration: 2000 }}` is just an object literal inside an expression tag — there is no special syntax.

---

## Summary

| Directive | Triggers On | Key Requirement |
|-----------|-------------|-----------------|
| `transition:` | enter or leave | none — bidirectional |
| `in:` | enter only | none |
| `out:` | leave only | none |
| `animate:` | keyed each reorder | element must be direct child of `{#each ... (key)}` |

| Animation | Built-in? | Notes |
|-----------|-----------|-------|
| `flip` | yes | uses FLIP technique on `from`/`to` rects |
| `whizz` (custom) | no | example: distance-aware duration |

See [references/in-out-reference.md](../references/in-out-reference.md) and [references/animations-reference.md](../references/animations-reference.md).