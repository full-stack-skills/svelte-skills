# TypeScript Reference

Deep dive into TypeScript integration with Svelte 5.

---

## `$props` Type Inference

### Basic Type Annotation

```svelte
<script lang="ts">
  let { name, age }: { name: string; age: number } = $props();
</script>
```

### With Optionals and Defaults

```svelte
<script lang="ts">
  interface Props {
    name: string;
    age?: number;
    isActive?: boolean;
  }

  let {
    name,
    age = 18,
    isActive = false
  }: Props = $props();
</script>
```

### With Callbacks

```svelte
<script lang="ts">
  interface Props {
    onClick?: (event: MouseEvent) => void;
    onSubmit?: (data: FormData) => void;
  }

  let { onClick, onSubmit }: Props = $props();
</script>
```

---

## Snippet Type Signature

### Importing Snippet Type

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';
</script>
```

### Basic Snippet Prop

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    children: Snippet;
  }

  let { children }: Props = $props();
</script>

{@render children()}
```

### Snippet with Parameters

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    items: string[];
    children: Snippet<[string, number]>;
  }

  let { items, children }: Props = $props();
</script>

<ul>
  {#each items as item, index}
    {@render children(item, index)}
  {/each}
</ul>
```

---

## Event Types

### Common DOM Event Types

| Event | Type | Example Handler |
|-------|------|-----------------|
| Click | `MouseEvent` | `(e: MouseEvent) => void` |
| Input | `Event` | `(e: Event) => void` |
| Keydown | `KeyboardEvent` | `(e: KeyboardEvent) => void` |
| Submit | `SubmitEvent` | `(e: SubmitEvent) => void` |
| Change | `Event` | `(e: Event) => void` |
| Focus | `FocusEvent` | `(e: FocusEvent) => void` |
| Drag | `DragEvent` | `(e: DragEvent) => void` |
| Wheel | `WheelEvent` | `(e: WheelEvent) => void` |
| Touch | `TouchEvent` | `(e: TouchEvent) => void` |

### Type-Safe Event Handlers

```svelte
<script lang="ts">
  function handleClick(e: MouseEvent) {
    // e.target is HTMLElement, not null
    const target = e.currentTarget as HTMLElement;
    console.log(target.dataset.id);
  }

  function handleInput(e: Event) {
    const target = e.target as HTMLInputElement;
    console.log(target.value);
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter' && e.ctrlKey) {
      console.log('Ctrl+Enter pressed');
    }
  }
</script>

<button onclick={handleClick}>Click</button>
<input oninput={handleInput} />
<textarea onkeydown={handleKeydown}></textarea>
```

---

## Component Types

### Component Props Extraction

```svelte
<script lang="ts">
  import type { Component, ComponentProps } from 'svelte';
  import MyComponent from './MyComponent.svelte';

  // Extract props type from component
  type MyProps = ComponentProps<MyComponent>;

  // Use the type
  const props: MyProps = { name: 'test' };
</script>

<MyComponent {...props} />
```

### Component Type for References

```svelte
<script lang="ts">
  import type { Component } from 'svelte';
  import Modal from './Modal.svelte';

  let modalRef: Component<{ open: boolean }> | null = $state(null);

  function openModal() {
    modalRef?.({ open: true });
  }
</script>

<Modal bind:this={modalRef} open={false} />
```

### Generic Components

```svelte
<script lang="ts" generics="TItem extends { id: string | number }">
  import type { Snippet } from 'svelte';

  interface Props {
    items: TItem[];
    children: Snippet<[TItem, number]>;
  }

  let { items, children }: Props = $props();
</script>

{#each items as item, index}
  {@render children(item, index)}
{/each}
```

### Generic Constraints

```svelte
<script lang="ts" generics="
  TItem extends object,
  TId extends keyof TItem
">
  let { items, idKey }: { items: TItem[]; idKey: TId } = $props();
</script>
```

---

## State Types

### `$state` with Type Annotation

```svelte
<script lang="ts">
  interface User {
    name: string;
    age: number;
  }

  let user = $state<User | null>(null);
  let items = $state<string[]>([]);
  let count = $state<number>(0);
</script>
```

### `$derived` Type Inference

```svelte
<script lang="ts">
  let { price, quantity }: { price: number; quantity: number } = $props();

  // Type is automatically inferred as number
  const total = $derived(price * quantity);
</script>
```

### `$effect` Type Safety

```svelte
<script lang="ts">
  let { userId }: { userId: string } = $props();

  $effect(() => {
    // userId is string inside effect
    console.log('User changed:', userId);
  });
</script>
```

---

## Module Types

### Importing Types

```svelte
<script lang="ts">
  // Type-only imports (not included in bundle)
  import type { Snippet, Component } from 'svelte';

  // Value imports (included in bundle)
  import { onMount } from 'svelte';
</script>
```

### Type-Only Exports

```ts
// types.ts
import type { Snippet, Component } from 'svelte';

export interface ButtonProps {
  variant?: 'primary' | 'secondary';
  children: Snippet;
}

export type { Snippet, Component };
```

---

## Utility Types

### Svelte Built-in Types

| Type | Description |
|------|-------------|
| `Snippet` | Snippet function type |
| `Component` | Svelte component type |
| `ComponentProps<T>` | Props of component T |
| `SvelteComponent` | Legacy component base type |
| `Action` | Svelte action type |
| `ActionReturn` | Return type of actions |

### Action Type Signature

```svelte
<script lang="ts">
  import type { Action } from 'svelte';

  const myAction: Action<HTMLElement, string> = (node, param) => {
    node.dataset.info = param;
    return {
      destroy() {
        delete node.dataset.info;
      }
    };
  };
</script>

<div use:myAction={'info'} />
```

---

## Common Patterns

### Form Handling Types

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

  function handleSubmit(e: SubmitEvent) {
    e.preventDefault();
    console.log(form);
  }
</script>

<form onsubmit={handleSubmit}>
  <input bind:value={form.name} name="name" required />
  <input bind:value={form.email} name="email" type="email" required />
  <label>
    <input bind:checked={form.agree} type="checkbox" />
    Agree to terms
  </label>
  <button type="submit">Submit</button>
</form>
```

### Async Data Types

```svelte
<script lang="ts">
  interface User {
    id: number;
    name: string;
    email: string;
  }

  let { userId }: { userId: number } = $props();

  let user = $state<User | null>(null);
  let loading = $state(false);
  let error = $state<string | null>(null);

  $effect(() => {
    loading = true;
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        user = data;
        loading = false;
      })
      .catch(err => {
        error = err.message;
        loading = false;
      });
  });
</script>
```

---

## Type Checking

### Strict Mode

Ensure `tsconfig.json` has strict mode enabled:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}
```

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Property 'x' does not exist` | Wrong type inference | Add explicit type annotation |
| `Type 'string \| undefined'` | Nullable without default | Provide default or use `?` |
| `Type 'never'` | Impossible assignment | Check logic/constraints |
| `Expected 2 arguments` | Missing snippet parameter | Pass required parameters |

---

## Best Practices

1. **Always use `lang="ts"`** on script tags for type checking
2. **Use `import type`** for type-only imports to avoid runtime overhead
3. **Define interfaces** for complex prop shapes
4. **Use strict mode** in TypeScript configuration
5. **Prefer inference** when types are obvious
6. **Explicit types** when dealing with unions or complex generics
7. **Use `ComponentProps<T>`** to extract types from existing components
