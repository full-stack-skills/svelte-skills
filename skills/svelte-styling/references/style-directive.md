# `style:` Directive Reference

A complete reference for the `style:` directive in Svelte 5.

## Overview

`style:` is a shorthand for setting individual inline styles on an element. It mirrors the `style="..."` attribute, but with one property per directive, expression support, and a special modifier for `!important`.

```svelte
<!-- Equivalent -->
<div style:color="red">A</div>
<div style="color: red">B</div>
```

## All Forms

### `style:property="literal"`

A static string value. Use for compile-time-known styles.

```svelte
<div style:background="#f5f5f5" style:padding="1rem">Static</div>
```

### `style:property={expression}`

Any JavaScript expression. The result is coerced to a string.

```svelte
<div style:color={myColor}>From state</div>
<div style:opacity={isActive ? 1 : 0.5}>Conditional</div>
<div style:transform={`translateX(${x}px)`}>Dynamic</div>
```

### `style:property` (shorthand)

When the property name matches a variable in scope, omit `=value`.

```svelte
<script>
  let color = $state('red');
  let width = $state('12rem');
</script>

<div style:color style:width>...</div>
```

The variable must be in scope (declared in `<script>`, a `$state`, a `$props`, etc.). It is read at render time and is reactive.

### `style:--custom-property`

Set a CSS custom property (variable). The name must start with `--`.

```svelte
<div style:--columns={columns} style:--gap="1rem">...</div>
```

The value is set as an inline style: `style="--columns: 3; --gap: 1rem;"`.

### `style:property|important`

Mark the style as `!important` using the `|important` modifier.

```svelte
<div style:color|important="red">Always red</div>
```

Compiled to `style="color: red !important;"`.

## Precedence

When `style:` directives are combined with a `style` attribute on the same element, the **directives win**.

```svelte
<!-- style: wins over style="" -->
<div style:color="red" style="color: blue">Red</div>

<!-- style: still wins, even when style="" uses !important -->
<div style:color="red" style="color: blue !important">Still red</div>
```

This is intentional. It lets you write a base `style="..."` and override specific properties with the shorthand.

## Reactivity

`style:` directives are reactive — the inline style updates as the bound expression changes.

```svelte
<script>
  let x = $state(0);
  $effect(() => {
    const id = setInterval(() => x = (x + 1) % 100, 50);
    return () => clearInterval(id);
  });
</script>

<div style:transform={`translateX(${x}px)`}>Moving</div>
```

Inside the compiled output, the directive becomes a per-property `style.setProperty(...)` call that runs whenever its dependencies change.

## Value Coercion

| Source value | What the browser sees |
|--------------|-----------------------|
| `'red'` | `'red'` |
| `42` | `'42'` (number — for length values, add the unit: `'42px'`) |
| `true` | `'true'` (boolean — almost always a bug) |
| `false` | `'false'` (boolean — almost always a bug) |
| `null` | property is removed (attribute omitted) |
| `undefined` | property is removed (attribute omitted) |
| `''` | property is set to empty string (different from null) |

For numeric length values, include the unit yourself — the directive does not infer it.

```svelte
<!-- WRONG: style="width: 10;" (invalid) -->
<div style:width={10}>

<!-- RIGHT -->
<div style:width="10px">              <!-- literal -->
<div style:width={`${10}px`}>         <!-- from expression -->
<div style:width="{10}px">             <!-- shorthand string interpolation -->
```

## Custom Property Value Coercion

Custom properties (CSS variables) are different from regular properties: every value is a string, and the browser only validates them when the variable is *consumed* via `var()`.

```svelte
<!-- This is fine: numbers are valid for custom properties -->
<div style:--columns={3}>

<!-- Even complex types are fine until used -->
<div style:--config={{ count: 3, gap: '1rem' }}>
```

In practice, keep custom property values as simple strings (`"3"`, `"1rem"`, `"#ff3e00"`). Passing objects or arrays works technically but is hard to consume in CSS.

## Custom Property Serialization

When you set `style:--name`, Svelte produces an inline `style` attribute with the property. There's no special handling for value type — whatever you put in is what the browser stores as the variable's value.

```svelte
<div style:--my-var="3">      <!-- style="--my-var: 3;" -->
<div style:--my-var="3px">    <!-- style="--my-var: 3px;" -->
<div style:--my-var="red">    <!-- style="--my-var: red;" -->
```

The `var()` consumer decides what to do with the value:

```css
.something {
  --my-var: 3;
  /* The variable is the string "3"; consumer must be a length */
  padding: calc(var(--my-var) * 1px);  /* OK */
  padding: var(--my-var);             /* INVALID: 3 is not a padding value */
}
```

## Multiple Styles

You can stack as many `style:` directives as you want on one element.

```svelte
<div
  style:color
  style:width="12rem"
  style:background-color={darkMode ? 'black' : 'white'}
  style:--columns={3}
  style:padding|important="1rem"
>
  Many styles
</div>
```

The directives are independent. The resulting `style` attribute is a single string, but Svelte maintains each property separately so updates to one don't clobber the others.

## Directive vs `style=""` Trade-offs

| Use `style:property` when | Use `style="..."` when |
|---------------------------|------------------------|
| You need reactivity (the value is bound to state) | The value is purely static and not a function of state |
| You want `style:` to win over a base `style="..."` | You need a string built from many parts |
| You want to pass an individual custom property | You want the readability of a multi-property declaration |
| You want `!important` via `|important` | You want `!important` baked into the string |

## Interaction with Component Composition

`style:` on a component element is treated as a regular HTML attribute, not a forwarded style. Svelte does not split the style across the component's children.

```svelte
<!-- These styles apply to <MyComponent>'s root, not its children -->
<MyComponent style:color="red" />
```

For deeper styling, use CSS custom properties:

```svelte
<MyComponent style:--text-color="red" />
```

```svelte
<!-- MyComponent.svelte -->
<style>
  .text { color: var(--text-color, black); }
</style>
```

## Common Mistakes

### Forgetting units on numeric values

```svelte
<!-- width: 10;  →  invalid CSS, no width set -->
<div style:width={10}>

<!-- Correct -->
<div style:width="{10}px">
```

### Using `style:` for a regular style shorthand

```svelte
<!-- This sets the 'background' shorthand to the literal string "red" -->
<div style:background="red">...</div>

<!-- vs. -->
<div style:background-color="red">...</div>
```

The directive does not parse CSS shorthand syntax. It writes the value verbatim to the `style` attribute, and the browser interprets the result.

### Confusing `style:` with `class:`

```svelte
<!-- Sets inline style -->
<div style:color="red">

<!-- Sets class name (deprecated in 5.16+; use class={{ ... }}) -->
<div class:cool={isCool}>
```

## Best Practices

- **Prefer `style:--var` over `style:property` for values that cross component boundaries.** Custom properties cascade naturally; inline styles don't.
- **Use `style:` for one or two reactive values that depend on state.** For more, use a class or a custom property.
- **Avoid `style:` for layout. Layout belongs in CSS.** Use a class (scoped or Tailwind) for the static part, and `style:` only for the few values that change at runtime.
- **Use the `|important` modifier sparingly.** It exists for cases where you truly need to override a higher-specificity rule, not as a general hammer.

## See also

- [examples/style-directive.md](../examples/style-directive.md) — practical examples
- [references/css-custom-properties.md](./css-custom-properties.md) — custom property patterns
- [references/specificity.md](./specificity.md) — when `style:` interacts with scoped and global rules
