# TypeScript Advanced Reference

Deep dive into the Svelte 5 + TypeScript integration: the `Component` type, generic components, wrapper components, `$state` typing, `svelte/elements`, and built-in DOM type augmentation.

---

## `<script lang="ts">` and Type-Only Features

Adding `lang="ts"` to a script tag turns on **type-only** TypeScript support — features that disappear when transpiled to JS (type annotations, interfaces, type aliases, generics).

**Supported (type-only):**

- Type annotations (`let x: number = 1`)
- Interface and type alias declarations
- Generics
- Type imports (`import type`)

**Not supported (need a preprocessor):**

- `enum` declarations
- Constructor parameter modifiers (`private`, `protected`, `public`) **with** initializers
- Anything beyond TC39 stage 4 (because Svelte's parser is Acorn)

For these, configure `vitePreprocess({ script: true })` or `svelte-preprocess`.

---

## Preprocessor Setup

### Vite (or SvelteKit)

```js
// svelte.config.js
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

export default {
  preprocess: vitePreprocess({ script: true })
};
```

### Rollup / Webpack

```bash
npm install -D typescript svelte-preprocess
```

```js
// rollup.config.js
import svelte from 'rollup-plugin-svelte';
import sveltePreprocess from 'svelte-preprocess';

export default {
  plugins: [svelte({ preprocess: sveltePreprocess() })]
};
```

---

## `tsconfig.json` Settings

| Option | Recommended | Why |
|--------|-------------|-----|
| `target` | `ES2015` or higher | Keep classes as classes; required for `$state` class fields |
| `isolatedModules` | `true` | Svelte/Vite compile files in isolation |
| `verbatimModuleSyntax` | `true` | Don't rewrite imports; let the bundler handle it |
| `strict` | `true` | Catch null/undefined issues |
| `module` | `ESNext` | Native ESM output |
| `moduleResolution` | `Bundler` | Matches Vite/SvelteKit |

Minimal `tsconfig.json`:

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

---

## Typing `$props`

Define an `interface Props` (or `type Props`) and annotate the destructure:

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    requiredProperty: number;
    optionalProperty?: boolean;
    snippetWithStringArgument: Snippet<[string]>;
    eventHandler: (arg: string) => void;
    [key: string]: unknown; // required for rest spread
  }

  let {
    requiredProperty,
    optionalProperty,
    snippetWithStringArgument,
    eventHandler,
    ...rest
  }: Props = $props();
</script>
```

The `[key: string]: unknown` index signature lets you spread arbitrary additional props onto an element. The optional signature is what makes `...rest` work without TypeScript complaining.

---

## Generic `$props`

The `generics` attribute on `<script>` declares type parameters:

```svelte
<script lang="ts" generics="Item extends { text: string }">
  interface Props {
    items: Item[];
    select(item: Item): void;
  }
  let { items, select }: Props = $props();
</script>

{#each items as item}
  <button onclick={() => select(item)}>{item.text}</button>
{/each}
```

Grammar is the same as the `<...>` of a generic function — multiple parameters, `extends`, and fallbacks are all allowed:

```svelte
<script lang="ts" generics="
  TItem extends { id: string | number },
  TKey extends keyof TItem = 'id'
">
  interface Props {
    items: TItem[];
    key?: TKey;
  }
  let { items, key }: Props = $props();
</script>
```

---

## Typing Wrapper Components

A wrapper component is one that renders a single underlying HTML element (e.g. `<Button>` → `<button>`). To forward all the element's attributes to the consumer, use the interface from `svelte/elements`:

### Dedicated interfaces (e.g. `HTMLButtonAttributes`)

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

### Generic `SvelteHTMLElements` index (for elements without a dedicated interface)

```svelte
<script lang="ts">
  import type { SvelteHTMLElements } from 'svelte/elements';
  let { children, ...rest }: SvelteHTMLElements['div'] = $props();
</script>

<div {...rest}>
  {@render children?.()}
</div>
```

### Extending with custom props

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

Available dedicated interfaces include `HTMLInputAttributes`, `HTMLAnchorAttributes`, `HTMLImageAttributes`, `HTMLFormAttributes`, `HTMLLabelAttributes`, etc. — any element exported from `svelte/elements`.

---

## The `Component` Type

`Component<Props>` is the type of a Svelte component constructor (i.e. the default export of a `.svelte` file). It replaces the legacy `SvelteComponent`.

### Constraining dynamic components

```svelte
<script lang="ts">
  import type { Component } from 'svelte';

  interface Props {
    DynamicComponent: Component<{ prop: string }>;
  }
  let { DynamicComponent }: Props = $props();
</script>

<DynamicComponent prop="foo" />
```

### Extracting props from a component

`ComponentProps<T>` returns the props type of a component:

```ts
import type { Component, ComponentProps } from 'svelte';
import MyComponent from './MyComponent.svelte';

function withProps<TComponent extends Component<any>>(
  component: TComponent,
  props: ComponentProps<TComponent>
) {}

// Type-checks
withProps(MyComponent, { foo: 'bar' });
```

### Constructor vs instance type

```svelte
<script lang="ts">
  import MyComponent from './MyComponent.svelte';

  let ctor: typeof MyComponent = MyComponent;        // constructor
  let inst: MyComponent | undefined = $state();     // instance
</script>

<MyComponent bind:this={inst} />
```

> In Svelte 4, components were of type `SvelteComponent` (the legacy class). In Svelte 5, use `Component` for the constructor and the imported class itself for instances.

---

## Typing `$state`

```ts
let count: number = $state(0);
```

If you don't give `$state` an initial value, the type is widened to `T | undefined`:

```ts
// @errors: 2322
let count: number = $state();
// Error: Type 'number | undefined' is not assignable to type 'number'
```

To assert the field will be defined before use, use an `as` cast — typical in classes:

```ts
class Counter {
  count = $state() as number;

  constructor(initial: number) {
    this.count = initial;
  }

  increment() {
    this.count += 1; // reactive
  }
}
```

> Reactive state in classes is shared between components automatically. Read the field through the instance to read the current value; assign through the instance to trigger updates.

For large API responses that you reassign rather than mutate, use `$state.raw` to skip the proxy:

```ts
interface User { /* ...lots of fields... */ }
let user = $state.raw<User | null>(null);
```

---

## `svelte/elements` — Enhancing Built-in DOM Types

`SvelteHTMLElements` is Svelte's type for every HTML element Svelte knows about. If a custom element, experimental attribute, or action-emitted custom event isn't typed, augment this module.

```ts
/// file: src/additional-svelte-typings.d.ts
import { HTMLButtonAttributes } from 'svelte/elements';

declare module 'svelte/elements' {
  // Add a new custom element
  export interface SvelteHTMLElements {
    'custom-button': HTMLButtonAttributes;
  }

  // Add a global attribute
  export interface HTMLAttributes<T> {
    globalattribute?: string;
  }

  // Add a button-specific attribute
  export interface HTMLButtonAttributes {
    veryexperimentalattribute?: string;
  }
}

export {}; // CRITICAL — forces the file to be a module; without it the
            // declarations become ambient and OVERRIDE the originals
```

Make sure the `.d.ts` is picked up by `tsconfig.json`:

```json
{
  "include": ["src/**/*"]
}
```

After augmenting, all Svelte code is fully typed for those additions.

If the attribute is a real, standardized HTML attribute that's missing from Svelte's typings, file an issue/PR at [sveltejs/svelte](https://github.com/sveltejs/svelte).

---

## `svelte-check` (CI)

`svelte-check` is the CLI counterpart to the VS Code extension:

```bash
npm install -D svelte-check
npx svelte-check --tsconfig ./tsconfig.json
```

Add to `package.json` scripts and CI. It reports:

- TS errors inside `.svelte` files
- A11y warnings (alt text, button types, label associations)
- Unused CSS selectors
- Unused script exports
- Invalid Svelte syntax

---

## Action Types

`Action<HTMLElement, ParameterType>`:

```svelte
<script lang="ts">
  import type { Action } from 'svelte';

  const tooltip: Action<HTMLElement, string> = (node, text) => {
    node.title = text;
    return {
      update(newText: string) { node.title = newText; },
      destroy() { node.title = ''; }
    };
  };
</script>

<button use:tooltip={'Save'}>Save</button>
```

`ActionReturn<ParameterType, Attributes>` describes the return shape if you need to attach custom attributes the action can update.

---

## Bindable Props

`$bindable()` opts a prop into two-way binding. Type the parent and the local alias the same way:

```svelte
<script lang="ts">
  interface Props {
    value: string;
  }
  let { value = $bindable('') }: Props = $props();
</script>

<input bind:value />
```

The default in `$bindable(default)` is used if the parent doesn't pass a value.

---

## Type-Only Imports

```svelte
<script lang="ts">
  import type { Snippet, Component } from 'svelte';
  import { onMount, flushSync } from 'svelte';
</script>
```

With `verbatimModuleSyntax: true`, TypeScript requires you to use `import type` for type-only imports, preventing accidental value imports of types.

---

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Property 'x' does not exist` | Wrong type inference | Add explicit type annotation |
| `Type 'string \| undefined' is not assignable to type 'string'` | Nullable without default | Add `?` or default value |
| `Type 'never'` | Impossible assignment | Check generics / constraints |
| `Expected 2 arguments` (to snippet) | Missing snippet parameter | Pass required parameters |
| `$state()` widening to `T \| undefined` | No initial value | Use `$state<T>(value)` or `as T` cast |
| `[key: string]: unknown` index signature required for `...rest` | TS requires index sig for rest spread | Add `[key: string]: unknown` to `Props` |

---

## Best Practices

1. **Always use `lang="ts"`** — Svelte's parser checks syntax; `svelte-check` checks types
2. **Use `import type`** — keep types out of the runtime bundle
3. **Define an `interface Props`** for non-trivial prop sets
4. **Use `strict: true`** in `tsconfig.json`
5. **Prefer type inference** when the type is obvious; explicit types for unions/generics
6. **Use `ComponentProps<T>`** to derive types from existing components
7. **Use `SvelteHTMLElements[T]`** for elements without a dedicated interface
8. **Run `svelte-check` in CI** to catch regressions early
