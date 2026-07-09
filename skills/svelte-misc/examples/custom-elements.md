# Custom Elements (Web Components) Examples

Compile Svelte components to native custom elements (Web Components) using `customElement` in `<svelte:options>`. Works with vanilla HTML/JS and most frameworks.

---

## 1. Basic Custom Element

Set the tag with `<svelte:options customElement="tag-name">` and the inner Svelte component is wrapped into a custom element class:

```svelte
<!-- MyElement.svelte -->
<svelte:options customElement="my-element" />

<script>
  let { name = 'world' } = $props();
</script>

<h1>Hello {name}!</h1>
```

Usage as a regular DOM element:

```html
<script type="module">
  import './MyElement.svelte'; // registers automatically
</script>

<my-element name="Svelte"></my-element>
```

> Inner components don't need a tag name. They keep working as Svelte components, and consumers can name them later with the static `element` property.

---

## 2. Accessing the Host Element with `$host`

Inside a custom element, use the `$host()` rune to access the host element (the `<my-element>` itself):

```svelte
<svelte:options customElement="tooltip-element" />

<script>
  import { $host } from 'svelte';

  let { text = 'hi' } = $props();

  function focus() {
    $host().focus();
  }
</script>

<button onclick={focus}>{text}</button>
```

---

## 3. Reading and Writing Props on the DOM

Props become DOM properties (and also attributes where possible):

```html
<script type="module">
  import './MyElement.svelte';
</script>

<my-element id="el"></my-element>
<script>
  const el = document.getElementById('el');

  // Read current value of the 'name' prop
  console.log(el.name);

  // Set new value, triggers re-render in shadow DOM
  el.name = 'everybody';
</script>
```

> You must list the prop explicitly in `customElement.props` (or as a destructured prop in `$props()`). `let props = $props()` alone is not enough — Svelte can't tell which keys to expose.

---

## 4. `customElement` Component Options

Full configuration object with `tag`, `shadow`, `props`, and `extend`:

```svelte
<svelte:options
  customElement={{
    tag: 'custom-element',
    shadow: {
      mode: import.meta.env.DEV ? 'open' : 'closed',
      clonable: true
    },
    props: {
      name: {
        reflect: true,
        type: 'Number',
        attribute: 'element-index'
      }
    },
    extend: (customElementConstructor) => {
      return class extends customElementConstructor {
        static formAssociated = true;

        constructor() {
          super();
          this.attachedInternals = this.attachInternals();
        }

        randomIndex() {
          this.elementIndex = Math.random();
        }
      };
    }
  }}
/>

<script>
  let { elementIndex, attachedInternals } = $props();

  function check() {
    attachedInternals.checkValidity();
  }
</script>
```

Key options:

- `tag` — element name; if set, the custom element is registered on import
- `shadow` — `"none"`, `"open"`, `"closed"`, or a `ShadowRootInit` object
- `props.<name>.attribute` — alternate HTML attribute name
- `props.<name>.reflect` — mirror value back to attribute
- `props.<name>.type` — `String | Boolean | Number | Array | Object`
- `extend` — wrap the generated class for custom lifecycle / `ElementInternals`

---

## 5. Lifecycle — Properties Set Before Mount

The inner Svelte component is created in the next tick after `connectedCallback`. Properties assigned before insertion are buffered and replayed on mount:

```js
// @noErrors
const el = document.createElement('my-element');
el.name = 'preset';   // buffered
document.body.appendChild(el); // mount happens next tick
// By next tick, name === 'preset' inside the Svelte component
```

Functions exported from the custom element, however, are only available **after** mount. To expose them earlier, define them inside `extend`:

```js
extend: (CustomElement) => {
  return class extends CustomElement {
    constructor() {
      super();
      this.attachedInternals = this.attachInternals();
    }
    // Available immediately after `new` — not just after mount
    randomIndex() {
      this.elementIndex = Math.random();
    }
  };
}
```

---

## 6. Slots → Children Rendered Eagerly

Custom elements use slots like a normal Web Component. In Svelte 5 the default is `<slot />` (and `{@render children()}`). Note: slotted content is rendered **eagerly** in the DOM (not lazily like in Svelte), and `<slot>` inside an `{#each}` does not duplicate slotted content.

```svelte
<!-- Card.svelte -->
<svelte:options customElement="my-card" />

<script>
  let { title = 'Card' } = $props();
</script>

<div class="card">
  <h3>{title}</h3>
  <slot />
</div>
```

```html
<my-card title="Info">
  <p>This is slotted content</p>
</my-card>
```

> The deprecated `let:` directive has no effect in custom elements — there is no way to pass data from the slot back to the parent.

---

## 7. Reflect Props to Attributes

By default prop changes do not reflect back to attributes. Enable per-prop with `reflect: true`:

```svelte
<svelte:options
  customElement={{
    tag: 'progress-bar',
    props: {
      value: { type: 'Number', reflect: true },
      label: { reflect: true }
    }
  }}
/>

<script>
  let { value = 0, label = '' } = $props();
</script>

<progress max="100" {value}></progress>
<span>{label}</span>
```

```html
<progress-bar value="50" label="Loading"></progress-bar>
<script>
  // value attribute updates as the prop changes
  document.querySelector('progress-bar').value = 75;
  // <progress-bar value="75"> now in the DOM
</script>
```

---

## 8. Typing Props with `type` for Coercion

By default, attribute values are strings. Declare the type for proper coercion both ways:

```svelte
<svelte:options
  customElement={{
    tag: 'rating',
    props: {
      stars: { type: 'Number' },
      filled: { type: 'Boolean' },
      label: { type: 'String' }
    }
  }}
/>

<script>
  let { stars = 0, filled = false, label = '' } = $props();
</script>
```

```html
<rating stars="5" filled label="Great"></rating>
<script>
  // stars is now number 5, filled is boolean true
  const r = document.querySelector('rating');
  console.log(r.stars + 1); // 6, not "51"
</script>
```

---

## 9. No Shadow DOM (`shadow: "none"`)

Without a shadow root, styles are not encapsulated and you can't use `<slot>`:

```svelte
<svelte:options
  customElement={{
    tag: 'inline-widget',
    shadow: 'none'
  }}
/>

<script>
  let { text = 'Hello' } = $props();
</script>

<span class="widget">{text}</span>
```

> Use this when you need the element to inherit global styles (e.g. for a CMS theme).

---

## 10. `ElementInternals` for Form-Associated Custom Elements

Combine `extend` with `attachInternals()` to participate in HTML forms (validity, form data, `formAssociated`):

```svelte
<svelte:options
  customElement={{
    tag: 'fancy-input',
    props: {
      value: { type: 'String', reflect: true }
    },
    extend: (Ctor) => class extends Ctor {
      static formAssociated = true;
      constructor() {
        super();
        /** @type {ElementInternals} */
        this.internals = this.attachInternals();
      }
      get form() { return this.internals.form; }
      get name() { return this.getAttribute('name'); }
      get type() { return this.localName; }
      get validity() { return this.internals.validity; }
      get validationMessage() { return this.internals.validationMessage; }
      get willValidate() { return this.internals.willValidate; }
      checkValidity() { return this.internals.checkValidity(); }
      reportValidity() { return this.internals.reportValidity(); }
      setValidity(flags, msg) { this.internals.setValidity(flags, msg); }
    }
  }}
/>

<script>
  let { value = $bindable('') } = $props();
</script>

<input bind:value />
```

---

## 11. Re-Defining the Tag Name from Outside

Leave the tag name off in `<svelte:options>` to keep the component private. Consumers can define it themselves with the static `element` property:

```svelte
<!-- AppWidget.svelte -->
<svelte:options customElement />

<script>
  let { name = 'App' } = $props();
</script>

<div>{name}</div>
```

```js
// @noErrors
import AppWidget from './AppWidget.svelte';

// Define with whatever tag name we like
customElements.define('app-widget', AppWidget.element);
```

---

## 12. Avoid `on*` Property/Attribute Names

Svelte interprets any prop or attribute starting with `on` as an event listener. Don't name your props `onsomething` — use a different name:

```svelte
<!-- BAD: <custom-element onvalue={true}> is treated as addEventListener('evalue', true) -->
<svelte:options customElement="bad" />
<script>
  let { onvalue } = $props(); // Don't do this
</script>
```

```svelte
<!-- GOOD -->
<svelte:options customElement="good" />
<script>
  let { onValue } = $props(); // not 'onvalue' — won't be treated as event
</script>
```

---

## Caveats Recap

- Styles are **encapsulated** (not merely scoped) inside the shadow root — global CSS won't reach them
- CSS is **inlined as a JS string** instead of extracted to a `.css` file
- Custom elements are **not SSR-friendly** — the shadow DOM is invisible until JS loads
- `<slot>` renders **eagerly** in the DOM (unlike lazy Svelte behavior)
- `let:` slot directive has **no effect**
- Older browsers need **polyfills**
- Svelte `context` works between regular Svelte components inside a custom element, but **not across** custom elements
- Don't name props `on*` — they're treated as event listeners
