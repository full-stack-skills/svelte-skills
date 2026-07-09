# svelte:window / svelte:document / svelte:body Reference

Comprehensive event and binding reference for the three "global listener" special elements. All three may only appear at the top level of a component — never inside a block or element.

```svelte
<svelte:window onevent={handler} bind:prop={value} />
<svelte:document onevent={handler} bind:prop={value} />
<svelte:body onevent={handler} use:action />
```

---

## svelte:window

Adds event listeners to the `window` object without worrying about cleanup, and binds to several window-level properties.

### Syntax

```svelte
<svelte:window onevent={handler} />
<svelte:window bind:prop={value} />
```

### Bindable Properties

| Property | Type | Readonly | Description |
|----------|------|----------|-------------|
| `innerWidth` | `number` | Yes | Viewport width in CSS pixels |
| `innerHeight` | `number` | Yes | Viewport height in CSS pixels |
| `outerWidth` | `number` | Yes | Browser window outer width (includes chrome) |
| `outerHeight` | `number` | Yes | Browser window outer height (includes chrome) |
| `scrollX` | `number` | **No** | Horizontal scroll position (writable) |
| `scrollY` | `number` | **No** | Vertical scroll position (writable) |
| `online` | `boolean` | Yes | Alias for `window.navigator.onLine` |
| `devicePixelRatio` | `number` | Yes | Current display's device pixel ratio |

> [!NOTE]
> `scrollX` and `scrollY` are the only writable bindings. The page is **not** scrolled to the initial value automatically (to avoid accessibility issues) — only subsequent changes to the bound variable will cause scrolling. If you have a legitimate reason to scroll on mount, call `scrollTo()` inside an `$effect`.

```svelte
<svelte:window bind:scrollY={y} />
```

### Event Listeners (all `window` events)

All standard `window` events can be listened to via the `on*` attribute. The most common ones:

#### Keyboard

- `onkeydown`, `onkeyup`, `onkeypress` (deprecated)

#### Mouse

- `onclick`, `oncontextmenu`, `ondblclick`
- `onmousedown`, `onmouseup`, `onmousemove`
- `onmouseover`, `onmouseout`, `onmouseenter`, `onmouseleave`
- `onmousewheel` (deprecated; use `onwheel`)

#### Touch

- `ontouchstart`, `ontouchmove`, `ontouchend`, `ontouchcancel`

#### Pointer

- `onpointerdown`, `onpointermove`, `onpointerup`
- `onpointercancel`, `onpointerover`, `onpointerout`, `onpointerenter`, `onpointerleave`
- `ongotpointercapture`, `onlostpointercapture`

#### Form / Input

- `onbeforeinput`, `oninput`, `onchange`
- `onsubmit`, `onreset`, `onfocus`, `onblur`
- `onselect`, `onselectionchange`

#### Scroll / Resize

- `onscroll`, `onresize`

#### Drag / Drop

- `ondrag`, `ondragstart`, `ondragend`
- `ondragenter`, `ondragover`, `ondragleave`, `ondrop`

#### Clipboard

- `oncopy`, `oncut`, `onpaste`

#### Media

- `oncanplay`, `oncanplaythrough`, `ondurationchange`
- `onemptied`, `onended`, `onloadeddata`, `onloadedmetadata`
- `onload`, `onerror` (window-level)
- `onpause`, `onplay`, `onplaying`, `onprogress`
- `onratechange`, `onseeked`, `onseeking`, `onstalled`, `onsuspend`
- `ontimeupdate`, `onvolumechange`, `onwaiting`

#### Focus

- `onfocus`, `onblur`

#### Other

- `onhashchange`, `onmessage`, `onmessageerror`
- `onoffline`, `ononline` (use `bind:online` instead, when possible)
- `onpagehide`, `onpageshow`
- `onpopstate`
- `onstorage`
- `onunload`, `onbeforeunload`
- `onwheel`

### Example: keyboard shortcut

```svelte
<script>
  function handleKeydown(event) {
    alert(`pressed the ${event.key} key`);
  }
</script>

<svelte:window onkeydown={handleKeydown} />
```

### Example: scroll position with `scrollTo` on mount

```svelte
<script>
  import { onMount } from 'svelte';
  let scroller;

  onMount(() => {
    scroller.scrollTo(0, 200); // explicitly scroll
  });
</script>

<div bind:this={scroller} style="height: 200vh">
  <svelte:window bind:scrollY={y} />
  <p>Y: {y}</p>
</div>
```

---

## svelte:document

Adds event listeners to the `document` object. Use this for events that **don't fire on `window`** — most commonly `visibilitychange`, `selectionchange`, and `readystatechange`. It also lets you use `attachments` on `document`.

### Syntax

```svelte
<svelte:document onevent={handler} {@attach someAttachment} />
<svelte:document bind:prop={value} />
```

### Bindable Properties (all readonly)

| Property | Type | Description |
|----------|------|-------------|
| `activeElement` | `Element \| null` | The currently focused element |
| `fullscreenElement` | `Element \| null` | The element currently in fullscreen mode |
| `pointerLockElement` | `Element \| null` | The element that has captured the pointer |
| `visibilityState` | `'visible' \| 'hidden'` | Document visibility (`document.visibilityState`) |

### Event Listeners (most useful `document` events)

#### Visibility / Lifecycle

- `onvisibilitychange` — page is hidden / shown (e.g. tab switch)
- `onreadystatechange` — `readyState` changes (`loading` → `interactive` → `complete`)
- `onDOMContentLoaded` — initial HTML parsed

#### Selection

- `onselectionchange` — text selection changes
- `onselectstart` — user starts a selection

#### Fullscreen

- `onfullscreenchange`
- `onfullscreenerror`

#### Pointer Lock

- `onpointerlockchange`
- `onpointerlockerror`

#### Drag

- `ondragstart`, `ondragend`, `ondragenter`, `ondragover`, `ondragleave`, `ondrop`

#### Other

- `oncopy`, `oncut`, `onpaste`
- `onfocus`, `onblur` (when focus moves into / out of the document)
- `onfullscreenchange`

> Some events (`visibilitychange`, `selectionchange`) only fire on `document` and **not** on `window` — these are the canonical use cases for `<svelte:document>`.

### Example: page visibility

```svelte
<svelte:document
  onvisibilitychange={handleVisibilityChange}
  {@attach someAttachment}
/>
```

### Example: track the focused element

```svelte
<script>
  let active = $state(null);
</script>

<svelte:document bind:activeElement={active} />

<p>Focused: {active?.tagName ?? 'none'}</p>
```

---

## svelte:body

Adds event listeners to `document.body` for events that do **not** fire on `window` — most notably `mouseenter` and `mouseleave` on the body. It also lets you apply Svelte [actions (`use:`)](https://svelte.dev/docs/svelte/use) to the `<body>` element.

### Syntax

```svelte
<svelte:body onevent={handler} use:someAction />
```

### Bindable Properties

None — `<svelte:body>` does not expose any bindings. Read body state via `<svelte:document>` or `document.body.*` directly.

### Event Listeners

`<svelte:body>` accepts any `body` event. The most useful ones:

- `onmouseenter`, `onmouseleave` — **the canonical use case** (do not fire on `window`)
- `onmousemove`, `onmouseover`, `onmouseout`
- `onload`
- `onfocus`, `onblur`
- `onresize` (rare on body, but supported)
- `onscroll` (rare on body)

### Example: track when the cursor enters / leaves the page

```svelte
<svelte:body
  onmouseenter={handleMouseenter}
  onmouseleave={handleMouseleave}
  use:someAction
/>
```

### Example: apply an action to `<body>`

```svelte
<script>
  function trapFocus(node) {
    // attach focus-trap behavior to the body
  }
</script>

<svelte:body use:trapFocus />
```

---

## Why three elements?

| Element | Targets | Most useful for |
|---------|---------|-----------------|
| `<svelte:window>` | `window` | Resize, scroll, keyboard, online status |
| `<svelte:document>` | `document` | `visibilitychange`, `selectionchange`, document-level focus |
| `<svelte:body>` | `document.body` | `mouseenter` / `mouseleave` on the body, body-level actions |

Pick the narrowest element that fires the event you care about.

---

## Common Pitfalls

1. **Top-level only.** None of these may appear inside a block, an element, or a snippet. They must be at the top of the component's template.
2. **No SSR `window` access.** The elements compile to no-ops on the server; the listener / binding is added on mount and removed on destroy. Don't rely on `bind:innerWidth` being set during SSR.
3. **`scrollX` / `scrollY` are the only writable window bindings.** Don't try to write to `innerWidth` etc. — they're readonly.
4. **Initial scroll position is not applied.** If you bind `scrollY` to `200`, the page will not scroll to 200 on mount. Use `scrollTo()` in an `$effect` for that.
5. **`mouseenter` / `mouseleave` do not bubble and do not fire on `window`.** Use `<svelte:body>` for them.
6. **Visibility and selection events only fire on `document`.** Use `<svelte:document>` for them.
7. **Document bindings are all readonly.** You cannot write to `visibilityState` to force-show a tab.
8. **Cleanup is automatic.** The listeners are removed when the component is destroyed, so you do not need to call `removeEventListener` yourself.
