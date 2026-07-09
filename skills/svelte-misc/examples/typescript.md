# TypeScript with Svelte Examples

Type-safe Svelte 5 patterns using TypeScript.

## 1. `$props` with Type Annotation

Basic typed props:

```svelte
<script lang="ts">
  interface Props {
    name: string;
    age: number;
    isActive?: boolean;
  }

  let { name, age, isActive = false }: Props = $props();
</script>

<p>Name: {name}, Age: {age}, Active: {isActive}</p>
```

---

## 2. `$props` with Snippet Type

Accepting children as snippets:

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    title: string;
    children: Snippet;
  }

  let { title, children }: Props = $props();
</script>

<header>
  <h1>{title}</h1>
  {@render children()}
</header>
```

---

## 3. Generic Components with Snippet

Generic component accepting typed snippets:

```svelte
<script lang="ts" generics="TItem">
  import type { Snippet } from 'svelte';

  interface Props {
    items: TItem[];
    children: Snippet<[TItem, number]>;
  }

  let { items, children }: Props = $props();
</script>

<ul>
  {#each items as item, index}
    {@render children(item, index)}
  {/each}
</ul>
```

```svelte
<!-- Usage -->
<GenericList items={users} let:item let:index>
  <li>{index}: {item.name}</li>
</GenericList>
```

---

## 4. Event Handler Types

Typing DOM event handlers:

```svelte
<script lang="ts">
  function handleClick(event: MouseEvent) {
    console.log('Clicked at', event.clientX, event.clientY);
  }

  function handleInput(event: Event) {
    const target = event.target as HTMLInputElement;
    console.log('Input value:', target.value);
  }

  function handleKeydown(event: KeyboardEvent) {
    if (event.key === 'Enter') {
      console.log('Enter pressed');
    }
  }
</script>

<button onclick={handleClick}>Click me</button>
<input oninput={handleInput} />
<input onkeydown={handleKeydown} />
```

---

## 5. ComponentRef with `bind:this`

Typing component references:

```svelte
<script lang="ts">
  import type { Component } from 'svelte';
  import Modal from './Modal.svelte';

  let modal: Component<{ open: boolean }> | null = $state(null);
</script>

<Modal bind:this={modal} open={false} />

<button onclick={() => modal?.({ open: true })}>
  Open Modal
</button>
```

---

## 6. `$bindable` for Two-Way Binding

Creating bindable props:

```svelte
<script lang="ts">
  interface Props {
    value: string;
  }

  let { value = $bindable() }: Props = $props();
</script>

<input bind:value />
```

```svelte
<!-- Usage with two-way binding -->
<Editable bind:value={text} />
<p>Current: {text}</p>
```

---

## 7. Event Dispatcher Types

Callback-based events with types:

```svelte
<script lang="ts">
  interface Props {
    onchange?: (value: number) => void;
    onsubmit?: (data: FormData) => void;
  }

  let { onchange, onsubmit }: Props = $props();

  function handleChange(event: Event) {
    const target = event.target as HTMLInputElement;
    onchange?.(parseInt(target.value, 10));
  }
</script>

<input type="range" onchange={handleChange} />
```

---

## 8. Importing Types

Using `import type` for type-only imports:

```svelte
<script lang="ts">
  import type { Snippet, Component } from 'svelte';
  import type { User } from './types';

  interface Props {
    user: User;
    children: Snippet;
  }

  let { user, children }: Props = $props();
</script>
```

---

## 9. `$state` with Type Inference

Type-safe reactive state:

```svelte
<script lang="ts">
  interface User {
    name: string;
    email: string;
  }

  let user = $state<User | null>(null);
  let items = $state<string[]>([]);
</script>

{#if user}
  <p>{user.name} ({user.email})</p>
{/if}
```

---

## 10. `$derived` with Type Safety

```svelte
<script lang="ts">
  let { price = 100, quantity = 1 } = $props();

  const total = $derived(price * quantity);
  const tax = $derived(total * 0.1);
  const grandTotal = $derived(total + tax);
</script>

<p>Subtotal: ${total.toFixed(2)}</p>
<p>Tax: ${tax.toFixed(2)}</p>
<p>Total: ${grandTotal.toFixed(2)}</p>
```

---

## 11. ComponentProps Utility Type

Extracting props from components:

```svelte
<script lang="ts">
  import type { Component, ComponentProps } from 'svelte';
  import Button from './Button.svelte';

  type ButtonProps = ComponentProps<Button>;

  const buttonProps: ButtonProps = {
    variant: 'primary',
    size: 'medium'
  };
</script>

<Button {...buttonProps}>Click me</Button>
```

---

## 12. SvelteAction Types

Typing custom actions:

```svelte
<script lang="ts">
  import type { Action } from 'svelte';

  const tooltip: Action<HTMLElement, string> = (node, text) => {
    node.title = text;
    return {
      destroy() {
        node.title = '';
      }
    };
  };
</script>

<button use:tooltip={'This is a tooltip'}>Hover me</button>
```
