# Custom Elements Reference

Complete reference for compiling Svelte components to custom elements (Web Components) via the `customElement` option in `<svelte:options>`. Covers the option surface, lifecycle, slot semantics, and caveats.

---

## Overview

Svelte components can be compiled to a `customElement` class that extends `HTMLElement`. The compiled element is a Web Component — it works with vanilla HTML/JS and any framework that supports custom elements (most of them do — see [custom-elements-everywhere.com](https://custom-elements-everywhere.com/)).

The minimum invocation:

```svelte
<svelte:options customElement="my-element" />

<script>
  let { name = 'world' } = $props();
</script>

<h1>Hello {name}!</h1>
<slot />
```

Once a custom element is imported, it's registered with `customElements.define()` automatically (when `tag` is set) — and can be used as a regular DOM element:

```html
<my-element name="Svelte">
  <p>This is some slotted content</p>
</my-element>
```

```js
// Reading/writing the prop on the DOM
const el = document.querySelector('my-element');
el.name = 'everybody';
```

---

## `$host()` Rune

Inside a custom element, use `$host()` to access the host element (the `<my-element>` itself). This is the Svelte equivalent of `this` in a custom element class method.

```svelte
<svelte:options customElement="tooltip-element" />

<script>
  import { $host } from 'svelte';
  let { text = 'tooltip' } = $props();
</script>

<button onclick={() => $host().dispatchEvent(new CustomEvent('action'))}>
  {text}
</button>
```

---

## Component Lifecycle

Custom elements are implemented as a **wrapper** around the inner Svelte component. The inner component has no awareness that it's a custom element.

**Mounting:**

1. The custom element is created.
2. The inner Svelte component is **not** yet created. Properties assigned to the custom element before insertion are buffered and applied on creation.
3. On `connectedCallback`, the Svelte component is created in the **next tick**.
4. Updates to the shadow DOM are batched and applied in the next tick — not synchronously. This means DOM moves that temporarily detach the element don't cause an unmount.

**Unmounting:**

- On `disconnectedCallback`, the inner Svelte component is destroyed in the **next tick**.

**Methods:**

- Methods exported from the Svelte component are only available after the inner component has mounted. To expose methods earlier, use the `extend` option in `customElement` and define them on the wrapping class.

---

## `customElement` Option Surface

The full configuration object:

```ts
interface CustomElementOptions {
  tag?: string;
  shadow?: 'none' | 'open' | 'closed' | ShadowRootInit;
  props?: Record<string, {
    attribute?: string;
    reflect?: boolean;
    type?: 'String' | 'Boolean' | 'Number' | 'Array' | 'Object';
  }>;
  extend?: (customElementConstructor: typeof HTMLElement) => typeof HTMLElement;
}
```

### `tag: string`

Optional. The element name. If set, the custom element is registered with `customElements.define()` automatically when the component module is imported.

```svelte
<svelte:options customElement={{ tag: 'my-counter' }} />
```

If `tag` is omitted (e.g. `customElement: true` or `customElement={...}` without `tag`), the component is private until the consumer defines it explicitly:

```js
import AppWidget from './AppWidget.svelte';
customElements.define('app-widget', AppWidget.element);
```

### `shadow`

Configures the shadow root. Default: `"open"`.

- `"none"` — no shadow root. Styles are **not** encapsulated, and `<slot>` is not available.
- `"open"` — `attachShadow({ mode: 'open' })`. External JS can read the shadow root via `el.shadowRoot`.
- `"closed"` — `attachShadow({ mode: 'closed' })`. External JS can't access the shadow root.
- `ShadowRootInit` object — passed to `attachShadow()`. Useful for `clonable`, `serializable`, `slotAssignment`, `delegatesFocus`.

```svelte
<svelte:options customElement={{
  tag: 'my-card',
  shadow: { mode: 'open', clonable: true, delegatesFocus: true }
}} />
```

### `props.<name>`

Customizes how a specific prop is handled.

| Property | Type | Default | Effect |
|----------|------|---------|--------|
| `attribute` | `string` | lowercase prop name | Override the HTML attribute name |
| `reflect` | `boolean` | `false` | Mirror prop value back to the attribute |
| `type` | `'String' \| 'Boolean' \| 'Number' \| 'Array' \| 'Object'` | `'String'` | Coerce attribute value to this type when reading/writing |

```svelte
<svelte:options customElement={{
  tag: 'rating',
  props: {
    stars: { type: 'Number' },
    filled: { type: 'Boolean' },
    label: { reflect: true, attribute: 'data-label' }
  }
}} />
```

Unlisted properties use defaults: `attribute` is the lowercase name, `reflect: false`, `type: 'String'`.

> You **must** list the prop explicitly (or destructure it in `$props()`) for it to be exposed. `let props = $props()` doesn't tell Svelte which keys to expose.

### `extend`

A function that takes the generated custom element class and returns a (sub)class. Used for advanced lifecycle work or to add `ElementInternals` for form association.

```svelte
<svelte:options customElement={{
  tag: 'fancy-input',
  extend: (Ctor) => class extends Ctor {
    static formAssociated = true;

    constructor() {
      super();
      this.internals = this.attachInternals();
    }

    checkValidity() { return this.internals.checkValidity(); }
  }
}} />
```

> TypeScript in `extend` is supported with limitations: set `lang="ts"` on one of the scripts in the file, and only use [erasable syntax](https://www.typescriptlang.org/tsconfig/#erasableSyntaxOnly) — preprocessors are not applied.

---

## Slots

Custom elements use native slot semantics. Slotted content is **rendered eagerly** in the DOM (unlike Svelte's normal lazy rendering) and `<slot>` inside an `{#each}` does **not** duplicate the slotted content.

```svelte
<!-- Card.svelte -->
<svelte:options customElement="my-card" />
<script>
  let { title = 'Card' } = $props();
</script>

<div class="card">
  <h3>{title}</h3>
  <slot />
  <slot name="footer" />
</div>
```

```html
<my-card title="Info">
  <p>Default slot</p>
  <p slot="footer">Footer slot</p>
</my-card>
```

The `let:` directive has no effect on slotted content in custom elements — there is no parent-to-slot data flow.

---

## Caveats and Limitations

1. **Style encapsulation** — Styles in `<style>` are encapsulated in the shadow root, not just scoped. Global CSS in a `global.css` will not apply. `:global(...)` is ignored inside the shadow.

2. **Styles inlined as JS** — CSS is bundled as a JavaScript string, not extracted to a `.css` file. This means a Svelte custom element is self-contained but cannot share CSS via external stylesheets.

3. **SSR unfriendly** — Custom elements depend on the shadow DOM, which is invisible until JS loads. Server-rendering doesn't help. The first paint will be empty.

4. **Eager slotted content** — `<slot>` always renders its children, even when inside an `{#if false}`. Similarly, `<slot>` inside an `{#each}` does not multiply the slotted content.

5. **`let:` doesn't work** — The deprecated `let:` slot directive has no effect in custom elements.

6. **Polyfills** — Older browsers (pre-2020) need a polyfill like `@webcomponents/webcomponentsjs`.

7. **Context doesn't cross custom element boundaries** — Svelte `context` works between regular Svelte components inside a custom element, but you cannot `setContext` on a parent custom element and `getContext` from a child custom element.

8. **Avoid `on*` prop/attribute names** — Any prop or attribute starting with `on` is treated as an event listener. `<my-el onvalue={true}>` becomes `addEventListener('evalue', true)`, not a prop assignment. Rename your props.

9. **Svelte 4 changes** — In Svelte 4, the `tag` shorthand is deprecated; use `customElement`. The property update timing changed slightly (next tick after `connectedCallback`, batched updates).

10. **Inner component lifecycle** — The Svelte component is created/destroyed in the next tick, not synchronously. This batches updates and prevents unwanted unmounts during DOM moves.

---

## Choosing a Shadow Root Mode

| Mode | When to use |
|------|-------------|
| `open` (default) | Standard cases where the host app might want to inspect the shadow root |
| `closed` | Strong encapsulation; consumers shouldn't touch internals |
| `none` | When the element must inherit global styles (CMS themes, plain HTML embedding) |

---

## ElementInternals (Form Association)

To make a custom element participate in `<form>` submission, set `formAssociated = true` and use `attachInternals()`:

```svelte
<svelte:options customElement={{
  tag: 'fancy-input',
  extend: (Ctor) => class extends Ctor {
    static formAssociated = true;
    constructor() { super(); this.internals = this.attachInternals(); }
    get form() { return this.internals.form; }
    get validity() { return this.internals.validity; }
    get validationMessage() { return this.internals.validationMessage; }
    get willValidate() { return this.internals.willValidate; }
    checkValidity() { return this.internals.checkValidity(); }
    reportValidity() { return this.internals.reportValidity(); }
    setValidity(flags, msg) { this.internals.setValidity(flags, msg); }
  }
}} />
```

The element then behaves like a native form control — participates in `form.submit()`, `form.elements`, validation, and reset.
