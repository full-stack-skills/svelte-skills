# `use:` Action Examples

Actions are functions invoked when an element is mounted. In Svelte 5.29+ prefer [`{@attach ...}`](#) (see [svelte-lifecycle skill](../../svelte-lifecycle/)) for better dependency tracking, but `use:` is still the canonical imperative-API pattern for libraries.

> [!NOTE]
> In Svelte 5.29 and newer, consider using [attachments](@attach) instead, as they are more flexible and composable.

---

## 1. Basic Action (No Parameters)

```svelte
<script>
  /** @type {import('svelte/action').Action} */
  function autofocus(node) {
    node.focus();
  }
</script>

<!-- focuses on mount -->
<input use:autofocus />
```

The action is called **once** when the element mounts (not during SSR, not on parameter changes).

---

## 2. Action With Parameters

```svelte
<script>
  /** @type {import('svelte/action').Action<HTMLInputElement, string>} */
  function tooltip(node, text) {
    const tip = document.createElement('div');
    tip.textContent = text;
    tip.className = 'tooltip';
    document.body.appendChild(tip);

    const place = (e) => {
      tip.style.left = `${e.pageX + 10}px`;
      tip.style.top = `${e.pageY + 10}px`;
    };

    node.addEventListener('mousemove', place);
    node.addEventListener('mouseleave', () => tip.remove());

    return {
      destroy() {
        tip.remove();
        node.removeEventListener('mousemove', place);
      }
    };
  }
</script>

<button use:tooltip={'Save (Ctrl+S)'}>Save</button>
```

The parameter (here `'Save (Ctrl+S)'`) is passed positionally. **It does not cause re-execution** when changed — use `$effect` inside the action if you need to react to parameter changes.

---

## 3. Action Returning Update/Destroy (Legacy Form)

Before `$effect`, actions returned `{ update, destroy }`. This still works:

```svelte
<script>
  /** @type {import('svelte/action').Action<HTMLDivElement, number>} */
  function colorize(node, color) {
    node.style.color = color;

    return {
      update(newColor) {
        node.style.color = newColor;
      },
      destroy() {
        node.style.color = '';
      }
    };
  }
</script>

<div use:colorize={'red'}>Hello</div>
<!-- later: <div use:colorize={dynamicColor}>Hello</div> -->
```

Prefer the `$effect`-based form below for new code.

---

## 4. Action Using `$effect` for Cleanup (Recommended)

```svelte
<script>
  /** @type {import('svelte/action').Action<HTMLElement, number>} */
  function measure(node, threshold = 0) {
    $effect(() => {
      const observer = new ResizeObserver((entries) => {
        const { width, height } = entries[0].contentRect;
        if (width > threshold && height > threshold) {
          console.log('large enough');
        }
      });
      observer.observe(node);

      return () => observer.disconnect();
    });
  }
</script>

<div use:measure={300}>Watched</div>
```

The cleanup function returned from `$effect` runs when the element unmounts (or when the action re-runs).

---

## 5. Typed Action (Three Type Arguments)

`Action<Node, Parameter, Events>`:

```svelte
<script lang="ts">
  import type { Action } from 'svelte/action';

  // Action that emits custom events: 'swipeleft' and 'swiperight'
  const gestures: Action<HTMLDivElement, undefined, {
    onswipeleft: (e: CustomEvent<{ direction: 'left' }>) => void;
    onswiperight: (e: CustomEvent<{ direction: 'right' }>) => void;
  }> = (node) => {
    let startX = 0;

    function down(e: PointerEvent) {
      startX = e.clientX;
      node.setPointerCapture(e.pointerId);
    }
    function up(e: PointerEvent) {
      const dx = e.clientX - startX;
      if (Math.abs(dx) > 50) {
        const detail = { direction: dx > 0 ? 'right' : 'left' } as const;
        node.dispatchEvent(new CustomEvent(dx > 0 ? 'swiperight' : 'swipeleft', { detail }));
      }
    }

    node.addEventListener('pointerdown', down);
    node.addEventListener('pointerup', up);

    return {
      destroy() {
        node.removeEventListener('pointerdown', down);
        node.removeEventListener('pointerup', up);
      }
    };
  };
</script>

<div use:gestures onswipeleft={next} onswiperight={prev}>
  swipe me
</div>
```

The third type parameter (`Events`) makes `onswipeleft` / `onswiperight` type-check in the template.

---

## 6. Reading Action Parameter Changes With `$effect.pre`

Use `$effect.pre` to read current parameter *before* DOM updates, or `$effect` to react after:

```svelte
<script>
  /**
   * @type {import('svelte/action').Action<HTMLInputElement, number>}
   */
  function debounce(node, delay) {
    let timer;

    $effect(() => {
      // delay is captured by the effect — runs again when it changes
      const handler = () => {
        clearTimeout(timer);
        timer = setTimeout(() => {
          node.dispatchEvent(new CustomEvent('debounced'));
        }, delay);
      };
      node.addEventListener('input', handler);

      return () => {
        node.removeEventListener('input', handler);
        clearTimeout(timer);
      };
    });
  }
</script>

<input use:debounce={300} ondebounced={save} />
```

Each time `delay` changes, the previous effect cleanup runs (clears timer, removes listener), then a new effect runs.

---

## 7. Reusable Action Library Pattern

```js
// lib/actions/clickOutside.js
export function clickOutside(node, callback) {
  function handle(e) {
    if (node && !node.contains(e.target) && !e.defaultPrevented) {
      callback(e);
    }
  }

  document.addEventListener('click', handle, true);

  return {
    destroy() {
      document.removeEventListener('click', handle, true);
    }
  };
}

// Used in many components
```

```svelte
<script>
  import { clickOutside } from '$lib/actions/clickOutside.js';

  let open = $state(false);
</script>

{#if open}
  <div use:clickOutside={() => (open = false)}>
    Dropdown
  </div>
{/if}
```

---

## 8. Action That Exposes Custom Events

```svelte
<script>
  /**
   * Long-press action that fires a `longpress` CustomEvent after hold.
   * @type {import('svelte/action').Action<HTMLButtonElement, number, {
   *   onlongpress: (e: CustomEvent) => void;
   * }>}
   */
  function longpress(node, ms = 500) {
    let timer;

    function start() {
      timer = setTimeout(() => {
        node.dispatchEvent(new CustomEvent('longpress'));
      }, ms);
    }
    function cancel() {
      clearTimeout(timer);
    }

    node.addEventListener('pointerdown', start);
    node.addEventListener('pointerup', cancel);
    node.addEventListener('pointerleave', cancel);

    return {
      destroy() {
        cancel();
        node.removeEventListener('pointerdown', start);
        node.removeEventListener('pointerup', cancel);
        node.removeEventListener('pointerleave', cancel);
      }
    };
  }
</script>

<button use:longpress={800} onlongpress={() => console.log('held')}>
  Hold me
</button>
```

---

## 9. Generic Action (Reusable Across Elements)

```svelte
<script lang="ts" generics="T extends HTMLElement">
  import type { Action } from 'svelte/action';

  /**
   * Track which element the cursor entered last and report its index.
   * @type {Action<T, number, { onenter: (e: CustomEvent<{ index: number }>) => void }>}
   */
  function reportIndex(node, index) {
    const onenter = () =>
      node.dispatchEvent(new CustomEvent('enter', { detail: { index } }));
    node.addEventListener('pointerenter', onenter);
    return {
      destroy() {
        node.removeEventListener('pointerenter', onenter);
      }
    };
  }
</script>

{#each items as item, i}
  <li use:reportIndex={i} onenter={(e) => console.log(e.detail.index)}>
    {item}
  </li>
{/each}
```

---

## 10. Action That Modifies Attributes

```svelte
<script>
  /**
   * @type {import('svelte/action').Action<HTMLImageElement, { src: string; alt: string }>}
   */
  function setSrc(node, params) {
    node.src = params.src;
    node.alt = params.alt;
  }
</script>

<img use:setSrc={{ src: '/cat.png', alt: 'A cat' }} />
```

> Tip: most attribute setting is more idiomatic with Svelte's built-in `<img src={...} alt={...}>` — reach for actions when you need imperative lifecycle behavior (event listeners, observers, third-party widgets).

---

## 11. Action With DOM Cleanup Needs

```svelte
<script>
  /**
   * @type {import('svelte/action').Action<HTMLDivElement, string>}
   */
  function portal(node, targetSelector = 'body') {
    const target = document.querySelector(targetSelector);
    if (!target) return;
    target.appendChild(node);

    return {
      destroy() {
        if (node.parentNode === target) {
          target.removeChild(node);
        }
      }
    };
  }
</script>

<div use:portal={'#modal-root'}>
  Rendered into #modal-root
</div>
```

---

## 12. Calling an Action Multiple Times on Same Element

Only **one** `use:` action of the same name can target an element. Chain effects by composing inside a single action:

```svelte
<script>
  import { clickOutside } from '$lib/actions/clickOutside.js';
  import { trapFocus } from '$lib/actions/trapFocus.js';

  /**
   * Combine two actions. Returns the merged action.
   * @type {import('svelte/action').Action<HTMLDivElement, () => void>}
   */
  function combined(node, onClickOutside) {
    const a = clickOutside(node, onClickOutside);
    const b = trapFocus(node);

    return {
      destroy() {
        a.destroy?.();
        b.destroy?.();
      }
    };
  }
</script>

<div use:combined={() => (open = false)}>
  Modal content
</div>
```

---

## Summary

| Pattern | When |
|---------|------|
| `use:action` (no params) | One-shot setup on mount |
| `use:action={param}` | Pass config to action |
| Return `{ update, destroy }` | Legacy / simpler API |
| Use `$effect` inside action | Modern, captures param changes |
| Add custom events with `CustomEvent` | Communicate from action to parent |
| Type with `Action<Node, Param, Events>` | Full type safety |

See [references/use-action-reference.md](../references/use-action-reference.md) for the complete API contract.