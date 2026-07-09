# Dynamic Components Reference

Complete reference for `svelte:element`, `{#key}`, `{@html}`, `{@debug}`, and `{@const}` in Svelte 5.

---

## `svelte:element`

Dynamically renders an HTML element or component based on a variable.

### Syntax

```svelte
<svelte:element this={tagOrComponent} />
<svelte:element this={tagOrComponent} {...props} />
```

### `svelte:element` vs `svelte:component`

| Aspect | `svelte:element` | `svelte:component` |
|--------|------------------|-------------------|
| **Purpose** | HTML elements only | Svelte components only |
| **Performance** | Lighter weight | Slightly heavier |
| **Use case** | Tag switching (div/span/h1) | Component swapping |
| **Added in** | Svelte 4 | Svelte 3 |

> **Svelte 5 Note**: `svelte:component` is deprecated. Use `svelte:element this={Component}` for components.

### Basic Examples

```svelte
<!-- Dynamic HTML tag -->
<script>
  let tag = 'h1';
</script>
<svelte:element this={tag}>Hello</svelte:element>

<!-- Dynamic Svelte component -->
<script>
  import First from './First.svelte';
  import Second from './Second.svelte';
  let Current = First;
</script>
<svelte:element this={Current} />
```

### With Props

```svelte
<script>
  import Alert from './Alert.svelte';

  let type = 'warning';
  const alerts = {
    info: Alert,
    warning: Alert,
    error: Alert
  };
</script>

<svelte:element
  this={alerts[type]}
  message="Something happened"
  {type}
/>
```

### Common Use Cases

1. **Heading levels**: `<DynamicHeading tag="h2">`
2. **Typography components**: `<TextElement tag="paragraph">`
3. **List items**: `<svelte:element this={ordered ? 'ol' : 'ul'}>`
4. **Form elements**: `<svelte:element this={inputType} />`

---

## `{#key}` Block

Destroys and recreates a block when its value changes.

### Syntax

```svelte
{#key value}
  <content />
{/key}
```

### Behavior

1. When `value` changes, Svelte unmounts the previous content
2. Svelte mounts new content with the updated value
3. Triggers animations/transitions if applied

### Use Cases

| Scenario | Example |
|----------|---------|
| **Form reset** | `{#key userId} <UserForm /> {/key}` |
| **Page transitions** | `{#key $page.url.pathname} <Page /> {/key}` |
| **Remounting components** | `{#key count} <Counter /> {/key}` |
| **Animation restart** | `{#key id} <AnimatedGraph data={newData} /> {/key}` |

### With Transitions

```svelte
{#key value}
  <div transition:fade>
    <Component />
  </div>
{/key}
```

### Key Points

- **Use when**: You need to force remounting (animations, form resets)
- **Avoid when**: Simple re-rendering suffices (use `$derived` instead)
- The key value must be primitive or object reference that changes

---

## `{@html}`

Renders raw HTML strings as actual DOM elements.

### Syntax

```svelte
{@html expression}
```

### Security Model

> **Warning**: Never use `{@html}` with unsanitized user input.

```svelte
<!-- UNSAFE - Don't do this -->
{@html userProvidedContent}

<!-- SAFE - Sanitize first -->
<script>
  import DOMPurify from 'dompurify';
  const safe = $derived(DOMPurify.sanitize(userContent));
</script>
{@html safe}
```

### Use Cases

| Use Case | Example |
|----------|---------|
| **Markdown rendering** | `{@html marked.parse(content)}` |
| **Rich text editor output** | `{@html editor.getHTML()}` |
| **Server-rendered HTML** | `{@html cmsContent}` |
| **SVG manipulation** | `{@html svgString}` |

### Best Practices

1. Always sanitize HTML before rendering
2. Use libraries like DOMPurify for sanitization
3. Consider using a sandboxed iframe for untrusted content
4. Avoid `{@html}` when possible; use proper templating

---

## `{@debug}`

Logs variables to the console during development.

### Syntax

```svelte
{@debug variable}
{@debug var1, var2, var3}
{@debug 'label', variable}
```

### When It Triggers

The debugger activates when:

1. A watched variable changes value
2. The component is first mounted
3. You manually trigger from console

### Console Output Format

```
{source file}:{line}
{
  variable: <value>,
  another: <value>
}
```

### Examples

```svelte
<!-- Single variable -->
{@debug count}

<!-- Multiple variables -->
{@debug count, name, user}

<!-- With label -->
{@debug 'state', count, items}
```

### Tips

- Remove `{@debug}` before production
- Use conditional debug: `{#if dev}{@debug data}{/if}`
- Combine with browser DevTools for better debugging

---

## `{@const}`

Defines a constant local to a template block.

### Syntax

```svelte
{@const name = expression}
```

### Scope

Constants are scoped to:
- The current block
- Child blocks (snippets, if blocks, each blocks)
- Do NOT leak outside the block

### Use Cases

| Scenario | Example |
|----------|---------|
| **Derived values in templates** | `{@const fullName = first + ' ' + last}` |
| **Loop calculations** | `{@const idx = items.indexOf(item)}` |
| **Conditional styling** | `{@const isActive = index === selected}` |
| **Repeated expressions** | `{@const tax = price * 0.1}` |

### Examples

```svelte
<!-- In each block -->
{#each items as item}
  {@const index = items.indexOf(item)}
  {@const isEven = index % 2 === 0}
  <div class:highlight={isEven}>{item}</div>
{/each}

<!-- In snippet -->
{#snippet row(item)}
  {@const isSelected = selected?.id === item.id}
  <tr class:selected={isSelected}>
    <td>{item.name}</td>
  </tr>
{/snippet}

<!-- In if block -->
{#if user}
  {@const initials = user.name.slice(0, 2).toUpperCase()}
  <Avatar {initials} />
{/if}
```

### Limitations

- Cannot use outside of template blocks
- Cannot reference other `{@const}` declarations in same block
- Not reactive - computed once when block evaluates

---

## Lifecycle Timing

Understanding when things execute:

| Feature | Timing | Notes |
|---------|--------|-------|
| `svelte:element` | Render | Re-evaluates when tag changes |
| `{#key}` | Mount/Unmount | Destroys old, creates new |
| `{@html}` | Render | Evaluates expression, inserts HTML |
| `{@debug}` | Change detection | Logs when watched vars change |
| `{@const}` | Block evaluation | Computed once per block entry |

---

## Common Patterns

### Dynamic Page Titles

```svelte
<svelte:element this={level}>
  <slot />
</svelte:element>
```

### Tab Switcher

```svelte
<script>
  import Tab1 from './Tab1.svelte';
  import Tab2 from './Tab2.svelte';

  const tabs = { home: Tab1, about: Tab2 };
  let current = $state('home');
</script>

<nav>
  {#each Object.keys(tabs) as tab}
    <button onclick={() => current = tab}>{tab}</button>
  {/each}
</nav>

{#key current}
  <svelte:element this={tabs[current]} transition:fade />
{/key}
```

### Reset Form on User Change

```svelte
<script>
  let { userId } = $props();
</script>

{#key userId}
  <form method="post" transition:fade>
    <input name="name" />
    <textarea name="bio"></textarea>
    <button>Save</button>
  </form>
{/key}
```
