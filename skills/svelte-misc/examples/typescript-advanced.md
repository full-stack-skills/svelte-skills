# TypeScript Advanced Examples

Advanced TypeScript patterns for Svelte 5: generics, the `Component` type, `svelte/elements`, built-in DOM type augmentation, preprocessor setup, and `$state` in classes.

---

## 1. `<script lang="ts">` and Preprocessor Setup

Add `lang="ts"` to use type-only features (annotations, interfaces, generics). Runtime-emitting features (enum, parameter properties with initializers, anything beyond TC39 stage 4) need a preprocessor:

```svelte
<script lang="ts">
  let name: string = 'world';
  function greet(name: string) { alert(`Hello, ${name}!`); }
</script>
<button onclick={(e: Event) => greet((e.target as HTMLElement).innerText)}>
  {name}
</button>
```

Configure `vitePreprocess({ script: true })` in `svelte.config.js` (Vite / SvelteKit). For Rollup / Webpack, install `typescript` + `svelte-preprocess`. Svelte's JS parser is Acorn — anything past stage 4 triggers preprocessor needs.

---

## 2. `tsconfig.json` Settings

```json
{
  "compilerOptions": {
    "target": "ES2015",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"]
}
```

Key flags: `target: "ES2015"`+ (keeps classes), `isolatedModules: true` (per-file compile), `verbatimModuleSyntax: true` (preserves imports), `strict: true` (null checks).

---

## 3. Typing `$props` with an Interface

The cleanest pattern for typed props is an `interface Props` (or `type Props`):

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    requiredProperty: number;
    optionalProperty?: boolean;
    snippetWithStringArgument: Snippet<[string]>;
    eventHandler: (arg: string) => void;
    [key: string]: unknown; // rest props
  }

  let {
    requiredProperty,
    optionalProperty,
    snippetWithStringArgument,
    eventHandler,
    ...everythingElse
  }: Props = $props();
</script>

<button onclick={() => eventHandler('clicked button')}>
  {@render snippetWithStringArgument('hello')}
</button>
```

The `[key: string]: unknown` index signature lets the rest spread work with arbitrary additional attributes.

---

## 4. Generic `$props` — Type-Safe List Component

The `generics` attribute on `<script>` declares type parameters — same syntax as between `<...>` of a generic function. Use `extends`, multiple parameters, and fallbacks freely.

```svelte
<script lang="ts" generics="Item extends { text: string }">
  interface Props {
    items: Item[];
    select(item: Item): void;
  }

  let { items, select }: Props = $props();
</script>

{#each items as item}
  <button onclick={() => select(item)}>
    {item.text}
  </button>
{/each}
```

Multiple generics with a constraint:

```svelte
<script lang="ts" generics="
  TItem extends { id: string | number },
  TKey extends keyof TItem = 'id'
">
  let { items, key }: { items: TItem[]; key?: TKey } = $props();
  const k = $derived((key ?? 'id') as TKey);
</script>

{#each items as item}
  <div data-key={item[k]}>{item.text}</div>
{/each}
```

---

## 5. Typing Wrapper Components

For components that wrap a native element, use the dedicated interfaces from `svelte/elements`:

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { HTMLButtonAttributes } from 'svelte/elements';
  let { children, ...rest }: HTMLButtonAttributes = $props();
</script>
<button {...rest}>{@render children?.()}</button>
```

For elements without a dedicated interface, use `SvelteHTMLElements['div']`. To add custom props, extend:

```svelte
<script lang="ts">
  import type { HTMLButtonAttributes } from 'svelte/elements';
  interface Props extends HTMLButtonAttributes { variant: 'primary' | 'secondary'; }
  let { variant, children, ...rest }: Props = $props();
</script>
<button {...rest} data-variant={variant}>{@render children?.()}</button>
```

---

## 6. The `Component` Type for Dynamic Components

`Component` is the Svelte 5 type for components (replacing `SvelteComponent`). Use it to constrain what components a prop can accept:

```svelte
<script lang="ts">
  import type { Component } from 'svelte';
  interface Props { DynamicComponent: Component<{ prop: string }>; }
  let { DynamicComponent }: Props = $props();
</script>
<DynamicComponent prop="foo" />
```

`ComponentProps<T>` extracts the props of an existing component:

```ts
import type { Component, ComponentProps } from 'svelte';
import MyComponent from './MyComponent.svelte';

function withProps<TComponent extends Component<any>>(
  component: TComponent,
  props: ComponentProps<TComponent>
) {}

// Type-errors if props don't match
withProps(MyComponent, { foo: 'bar' });
```

Constructor vs instance: `let ctor: typeof MyComponent = MyComponent;` and `let inst: MyComponent;` then `<MyComponent bind:this={inst} />`.

---

## 7. Typing `$state` in Classes (with `as` cast)

`$state` works in `.svelte.ts` classes. If you don't supply an initial value, TypeScript widens the type to include `undefined`:

```ts
// @errors: 2322
class Counter {
  // Error: Type 'number | undefined' is not assignable to 'number'
  count = $state<number>();
}
```

If the field will be assigned before use, cast with `as`:

```ts
class Counter {
  count = $state() as number;

  constructor(initial: number) {
    this.count = initial;
  }

  increment() {
    this.count += 1; // reactive — proxied
  }
}
```

> State declared in classes is **not** deeply reactive in the same way as in components. Access via the class instance triggers updates because reads happen inside component effects.

A typed reactive class — sharing reactivity between components:

```ts
// counter.svelte.ts
export class Counter {
  count = $state(0);
  history = $state<number[]>([]);

  increment() {
    this.count += 1;
    this.history.push(this.count);
  }
}
```

```svelte
<!-- Display.svelte -->
<script lang="ts">
  import { Counter } from './counter.svelte';
  const counter = new Counter();
</script>

<button onclick={() => counter.increment()}>
  {counter.count} (history: {counter.history.length})
</button>
```

---

## 8. Enhancing Built-in DOM Types (Augmenting `svelte/elements`)

If you use a custom element, experimental attribute, or a custom event from an action, augment `svelte/elements`:

```ts
/// file: src/additional-svelte-typings.d.ts
import { HTMLButtonAttributes } from 'svelte/elements';

declare module 'svelte/elements' {
  // Add a new custom element
  export interface SvelteHTMLElements {
    'custom-button': HTMLButtonAttributes;
  }

  // Add a new global attribute available on every HTML element
  export interface HTMLAttributes<T> {
    globalattribute?: string;
  }

  // Add a new attribute to all buttons
  export interface HTMLButtonAttributes {
    veryexperimentalattribute?: string;
  }
}

export {}; // CRITICAL — without this, declarations become ambient and OVERRIDE instead of augment
```

Make sure the `.d.ts` is included in `tsconfig.json` (a `"include": ["src/**/*"]` pattern usually catches it).

Now usage is fully typed:

```svelte
<script lang="ts">
  let { children, ...rest } = $props();
</script>

<button {...rest} data-globalattribute="foo">
  {@render children?.()}
</button>

<custom-button veryexperimentalattribute="yes">Click</custom-button>
```

---

## 9. `svelte-check` for CI Integration

`svelte-check` is the CLI companion to the Svelte VS Code extension. Run it in CI to catch type errors and Svelte-specific issues (a11y warnings, unused CSS selectors, invalid syntax):

```bash
npm install -D svelte-check
npx svelte-check --tsconfig ./tsconfig.json
```

Add to `package.json`:

```json
{ "scripts": { "check": "svelte-check --tsconfig ./tsconfig.json" } }
```

In CI, run `npm run check` before `npm run build` / `npm test`.

---

## 10. Bindable Props with Type Safety

`$bindable()` opts a prop into two-way binding:

```svelte
<!-- TextField.svelte -->
<script lang="ts">
  interface Props { value: string; placeholder?: string; }
  let { value = $bindable(''), placeholder = '' }: Props = $props();
</script>
<input bind:value {placeholder} />
```

```svelte
<script lang="ts">
  let text = $state('initial');
</script>
<TextField bind:value={text} />
<p>{text}</p>
```

`$bindable(default)` provides the value used when the parent doesn't pass anything.

---

## 11. Action Type with Parameter

`Action<HTMLElement, ParameterType>` types custom actions so the parameter is checked at the use site:

```svelte
<script lang="ts">
  import type { Action } from 'svelte';

  const tooltip: Action<HTMLElement, string> = (node, text) => {
    node.title = text;
    return {
      update(newText: string) {
        node.title = newText;
      },
      destroy() {
        node.title = '';
      }
    };
  };
</script>

<button use:tooltip={'Click me'}>Hover me</button>
```

Action with a richer parameter object:

```svelte
<script lang="ts" generics="Params extends { color: string }">
  import type { Action } from 'svelte';

  const paint: Action<HTMLElement, Params> = (node, params) => {
    node.style.color = params.color;
    return {
      update(p) { node.style.color = p.color; }
    };
  };
</script>

<p use:paint={{ color: 'rebeccapurple' }}>Hello</p>
```

---

## 12. `import type` and Type-Only Exports

`import type` keeps type-only imports out of the runtime bundle:

```svelte
<script lang="ts">
  import type { Snippet, Component } from 'svelte'; // stripped at build
  import { onMount, flushSync } from 'svelte';        // kept in bundle
</script>
```

For shared types, put them in a dedicated `.ts` file with `import type` for types and `export type { ... }` for re-exports. `verbatimModuleSyntax: true` in `tsconfig.json` enforces this distinction.

---

## 13. Form State with Strict Types

```svelte
<script lang="ts">
  interface FormData { name: string; email: string; agree: boolean; }
  let form = $state<FormData>({ name: '', email: '', agree: false });
  let errors = $state<Partial<Record<keyof FormData, string>>>({});

  function handleSubmit(e: SubmitEvent) {
    e.preventDefault();
    if (!form.email.includes('@')) errors.email = 'Invalid email';
  }
</script>

<form onsubmit={handleSubmit}>
  <input bind:value={form.name} />
  <input bind:value={form.email} type="email" />
  <label><input bind:checked={form.agree} type="checkbox" /> Agree</label>
  {#if errors.email}<p class="err">{errors.email}</p>{/if}
  <button type="submit">Save</button>
</form>
```

---

## 14. Typing `bind:this` to Component Instances

```svelte
<script lang="ts">
  import type { Component } from 'svelte';
  import Modal from './Modal.svelte';

  let modal: Component<{ open: boolean }> | null = $state(null);
</script>

<Modal bind:this={modal} open={false} />
<button onclick={() => modal?.$set({ open: true })}>Open Modal</button>
```

For a strongly-typed instance, use the default export's instance type: `let modal: Modal | undefined = $state();`.
