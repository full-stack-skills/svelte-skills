# TypeScript Advanced Examples

Advanced TypeScript patterns for Svelte 5: generics, the `Component` type, `svelte/elements`, built-in DOM type augmentation, preprocessor setup, and `$state` in classes.

---

## 1. `<script lang="ts">` and Type-Only Features

Svelte natively supports TypeScript. Just add `lang="ts"` and use type-only features (type annotations, interfaces, generics). Features that compile to runtime code (enums, parameter properties, post-TC39-stage-4 syntax) require a preprocessor.

```svelte
<script lang="ts">
  let name: string = 'world';

  function greet(name: string) {
    alert(`Hello, ${name}!`);
  }
</script>

<button onclick={(e: Event) => greet((e.target as HTMLElement).innerText)}>
  {name}
</button>
```

> The Svelte compiler uses Acorn to parse JS. Anything beyond TC39 stage 4 (e.g. enum, `private` constructor modifiers) needs `vitePreprocess({ script: true })` or `svelte-preprocess`.

---

## 2. Preprocessor Setup with Vite (For Non-Type-Only TS)

When you need enums, parameter properties, or other TypeScript features that emit code, configure `vitePreprocess` in `svelte.config.js`:

```js
/// file: svelte.config.js
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

const config = {
  preprocess: vitePreprocess({ script: true })
};

export default config;
```

For non-Vite toolchains (Rollup, Webpack), use `svelte-preprocess`:

```bash
npm install -D typescript svelte-preprocess
```

```js
// rollup.config.js
import sveltePreprocess from 'svelte-preprocess';

export default {
  plugins: [
    svelte({
      preprocess: sveltePreprocess()
    })
  ]
};
```

---

## 3. `tsconfig.json` Settings for Svelte

A Svelte-friendly `tsconfig.json` baseline:

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
    "skipLibCheck": true,
    "allowJs": true,
    "checkJs": true
  },
  "include": ["src/**/*"]
}
```

Key flags:

- `target: "ES2015"` (or higher) — keeps classes as classes, not transpiled to functions
- `isolatedModules: true` — Svelte/Vite compile each file in isolation
- `verbatimModuleSyntax: true` — leaves imports intact for the bundler to handle
- `strict: true` — catch null/undefined issues early

---

## 4. Typing `$props` with an Interface

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

## 5. Generic `$props` — Type-Safe List Component

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

## 6. Typing Wrapper Components (Button Wrapping `<button>`)

For components that wrap a native element, use the dedicated interfaces from `svelte/elements`:

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { HTMLButtonAttributes } from 'svelte/elements';

  let { children, ...rest }: HTMLButtonAttributes = $props();
</script>

<button {...rest}>
  {@render children?.()}
</button>
```

For elements without a dedicated interface, use the `SvelteHTMLElements` index:

```svelte
<script lang="ts">
  import type { SvelteHTMLElements } from 'svelte/elements';

  let { children, ...rest }: SvelteHTMLElements['div'] = $props();
</script>

<div {...rest}>
  {@render children?.()}
</div>
```

Extending with custom props:

```svelte
<script lang="ts">
  import type { HTMLButtonAttributes } from 'svelte/elements';

  interface Props extends HTMLButtonAttributes {
    variant: 'primary' | 'secondary';
  }

  let { variant, children, ...rest }: Props = $props();
</script>

<button {...rest} data-variant={variant}>
  {@render children?.()}
</button>
```

---

## 7. The `Component` Type for Dynamic Components

`Component` is the Svelte 5 type for components (replacing `SvelteComponent`). Use it to constrain what components a prop can accept:

```svelte
<!-- Container.svelte -->
<script lang="ts">
  import type { Component } from 'svelte';

  interface Props {
    // Only components that accept a `prop: string` can be passed
    DynamicComponent: Component<{ prop: string }>;
  }

  let { DynamicComponent }: Props = $props();
</script>

<DynamicComponent prop="foo" />
```

Use `ComponentProps<T>` to extract the props of an existing component:

```ts
import type { Component, ComponentProps } from 'svelte';
import MyComponent from './MyComponent.svelte';

function withProps<TComponent extends Component<any>>(
  component: TComponent,
  props: ComponentProps<TComponent>
) {}

// Errors if props don't match
withProps(MyComponent, { foo: 'bar' });
```

The constructor/instance types:

```svelte
<script lang="ts">
  import MyComponent from './MyComponent.svelte';

  let componentConstructor: typeof MyComponent = MyComponent;
  let componentInstance: MyComponent;
</script>

<MyComponent bind:this={componentInstance} />
```

> In Svelte 4, components were of type `SvelteComponent` (legacy).

---

## 8. Typing `$state` in Classes (with `as` cast)

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

## 9. Enhancing Built-in DOM Types (Augmenting `svelte/elements`)

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

## 10. `svelte-check` for CI Integration

`svelte-check` is the CLI companion to the Svelte VS Code extension. Run it in CI to catch type errors and Svelte-specific issues:

```bash
# Install
npm install -D svelte-check

# Run
npx svelte-check --tsconfig ./tsconfig.json

# Add to package.json scripts
{
  "scripts": {
    "check": "svelte-check --tsconfig ./tsconfig.json",
    "check:watch": "svelte-check --tsconfig ./tsconfig.json --watch"
  }
}
```

In CI:

```yaml
# .github/workflows/ci.yml
- run: npm run check
- run: npm run build
- run: npm test
```

`svelte-check` reports:
- TypeScript errors inside `.svelte` files
- A11y warnings (missing `alt`, button types, etc.)
- Unused CSS selectors
- Unused script exports
- Invalid Svelte syntax

---

## 11. Bindable Props with Type Safety

`$bindable()` opts a prop into two-way binding. Type it the same way you type any other prop:

```svelte
<!-- TextField.svelte -->
<script lang="ts">
  interface Props {
    value: string;
    placeholder?: string;
  }

  let { value = $bindable(''), placeholder = '' }: Props = $props();
</script>

<input bind:value {placeholder} />
```

```svelte
<!-- Usage -->
<script lang="ts">
  let text = $state('initial');
</script>

<TextField bind:value={text} />
<p>{text}</p>
```

Bindable with a default fallback:

```svelte
<script lang="ts">
  interface Props {
    count?: number;
  }
  let { count = $bindable(0) }: Props = $props();
</script>

<button onclick={() => count++}>{count}</button>
```

---

## 12. Action Type with Parameter

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

## 13. `import type` and Type-Only Exports

Use `import type` to keep type-only imports out of the runtime bundle:

```svelte
<script lang="ts">
  // Type-only — stripped at build time
  import type { Snippet, Component } from 'svelte';

  // Value import — kept in bundle
  import { onMount, flushSync } from 'svelte';
</script>
```

In a separate types file:

```ts
// src/lib/types.ts
import type { Snippet, Component } from 'svelte';

export interface ButtonProps {
  variant?: 'primary' | 'secondary';
  size?: 'sm' | 'md' | 'lg';
  children: Snippet;
}

export type { Component };
```

> `verbatimModuleSyntax: true` in `tsconfig.json` enforces this distinction and prevents accidental value imports of types.

---

## 14. Form State with Strict Types

A common typed form pattern:

```svelte
<script lang="ts">
  interface FormData {
    name: string;
    email: string;
    agree: boolean;
  }

  let form = $state<FormData>({
    name: '',
    email: '',
    agree: false
  });

  let errors = $state<Partial<Record<keyof FormData, string>>>({});

  function handleSubmit(e: SubmitEvent) {
    e.preventDefault();
    // validate
    if (!form.email.includes('@')) errors.email = 'Invalid email';
  }
</script>

<form onsubmit={handleSubmit}>
  <input bind:value={form.name} />
  <input bind:value={form.email} type="email" />
  <label>
    <input bind:checked={form.agree} type="checkbox" />
    Agree
  </label>
  {#if errors.email}<p class="err">{errors.email}</p>{/if}
  <button type="submit">Save</button>
</form>
```

---

## 15. Typing `bind:this` to Component Instances

```svelte
<script lang="ts">
  import type { Component } from 'svelte';
  import Modal from './Modal.svelte';

  let modal: Component<{ open: boolean }> | null = $state(null);
</script>

<Modal bind:this={modal} open={false} />

<button onclick={() => modal?.$set({ open: true })}>
  Open Modal
</button>
```

For a strongly-typed instance, use the default export's instance type:

```svelte
<script lang="ts">
  import Modal from './Modal.svelte';

  let modal: Modal | undefined = $state();
</script>

<Modal bind:this={modal} />
```
