# `style:` Directive Examples

The `style:` directive is a shorthand for setting individual inline styles on an element. It mirrors the `style="..."` attribute but with one style per directive, scoped names, and expression support.

This file covers every form: simple properties, expressions, shorthand, multiple styles, `|important`, and CSS custom properties.

## Table of Contents

- [Basic `style:property` Shorthand](#basic-styleproperty-shorthand)
- [String Value (`style:color="red"`)](#string-value-stylecolorred)
- [Expression Value (`style:color={...}`)](#expression-value-stylecolor)
- [Identifier Shorthand (`style:color`)](#identifier-shorthand-stylecolor)
- [Multiple Styles on One Element](#multiple-styles-on-one-element)
- [CSS Custom Properties (`style:--var`)](#css-custom-properties-style--var)
- [`!important` via the `|important` Modifier](#important-via-the-important-modifier)
- [`style:` vs `style=""` Precedence](#style-vs-style-precedence)
- [Dynamic Style with `$state` and `$derived`](#dynamic-style-with-state-and-derived)
- [Computed / `calc()` Values](#computed--calc-values)
- [Setting Vendor-Prefixed Properties](#setting-vendor-prefixed-properties)
- [Custom Properties and `var()` Cascade](#custom-properties-and-var-cascade)
- [Common Mistakes](#common-mistakes)

## Basic `style:property` Shorthand

```svelte
<!-- All three are equivalent -->
<div style:color="red">A</div>
<div style="color: red">B</div>
<div style:color>C</div>     <!-- when `color` is a variable -->
```

## String Value (`style:color="red"`)

The right-hand side is a string literal. Use this for static values.

```svelte
<div style:background="#f5f5f5" style:padding="1rem">Static styles</div>
```

## Expression Value (`style:color={...}`)

Wrap any JavaScript expression in `{}`. The result is coerced to a string and used as the property value.

```svelte
<script>
  let myColor = $state('tomato');
  let isActive = $state(true);
</script>

<div style:color={myColor}>Color from state</div>
<div style:opacity={isActive ? 1 : 0.5}>Conditional opacity</div>
<div style:transform={`translateX(${x}px)`}>Dynamic transform</div>
```

Numbers are stringified; you may need to add units yourself for length values:

```svelte
<!-- WRONG: width ends up as the literal "10" (no unit) -->
<div style:width={10}>...</div>

<!-- RIGHT: include the unit -->
<div style:width="{10}px">...</div>
```

## Identifier Shorthand (`style:color`)

When the property name and the variable name are the same, omit the `=value` part.

```svelte
<script>
  let color = $state('red');
  let width = $state('12rem');
</script>

<!-- Equivalent to style:color={color} and style:width={width} -->
<div style:color style:width>...</div>
```

The variable must be in scope (declared in the `<script>` block, a `$state`, a `$props`, etc.).

## Multiple Styles on One Element

You can stack as many `style:` directives as you like.

```svelte
<div
  style:color
  style:width="12rem"
  style:background-color={darkMode ? 'black' : 'white'}
  style:padding
>
  Multiple styles
</div>
```

The directives are independent — no need to comma-separate or anything.

## CSS Custom Properties (`style:--var`)

The `style:` directive supports custom properties (CSS variables) by prefixing the name with `--`.

```svelte
<script>
  let columns = $state(3);
  let gap = $state('1rem');
</script>

<div style:--columns={columns} style:--gap={gap}>
  <p>Item 1</p>
  <p>Item 2</p>
  <p>Item 3</p>
</div>

<style>
  div {
    display: grid;
    grid-template-columns: repeat(var(--columns, 1), 1fr);
    gap: var(--gap, 0);
  }
</style>
```

### Why use `style:--var` over `style="--var: ..."`?

Both work. `style:--var` keeps the rest of the style attribute cleaner when you also need to set regular properties, and Svelte will treat the name as a custom property even before you set it (so TS tooling doesn't flag it).

### Custom property naming rules

- Must start with `--` (two dashes).
- Otherwise follows CSS custom property naming (kebab-case recommended).
- The directive does not validate the name; invalid names will be passed to the DOM as-is, where the browser will ignore them.

## `!important` via the `|important` Modifier

Use the `|important` modifier to mark a style as `!important`.

```svelte
<div style:color|important="red">Always red</div>
```

The compiled output is `style="color: red !important"`.

## `style:` vs `style=""` Precedence

When `style:` directives are combined with a `style` attribute on the same element, the **directives win**, even if the `style` attribute has `!important`.

```svelte
<!-- style: takes precedence; this text is red, not blue -->
<div style:color="red" style="color: blue">Red text</div>

<!-- style: still wins over !important in the style attribute -->
<div style:color="red" style="color: blue !important">Still red</div>
```

This is intentional: it lets you write a base style and override specific properties with the directive shorthand.

## Dynamic Style with `$state` and `$derived`

`style:` directives are reactive — they update as the bound expression changes.

```svelte
<script>
  let progress = $state(0);
  let hue = $derived(200 + progress * 1.2);
</script>

<input type="range" min="0" max="100" bind:value={progress} />

<div
  style:background-color={`hsl(${hue}, 80%, 60%)`}
  style:--progress="{progress}%"
>
  Preview
</div>

<style>
  div {
    --progress: 0%;
    width: 200px;
    height: 60px;
    background: linear-gradient(
      to right,
      currentColor 0%,
      currentColor var(--progress),
      transparent var(--progress)
    );
  }
</style>
```

## Computed / `calc()` Values

Use template strings or `calc()` to compose values.

```svelte
<script>
  let size = $state(16);
  let factor = $state(1.5);
</script>

<div style:font-size="{size}px">Literal units</div>
<div style:width="calc({size} * {factor} * 1rem)">Computed width</div>
```

`calc()` works because the resulting `style` string is just a CSS value; the browser interprets it.

## Setting Vendor-Prefixed Properties

The directive accepts any string as a property name, including vendor-prefixed ones.

```svelte
<div style:-webkit-line-clamp="3">Truncated text</div>
```

Svelte does not transform or validate the property name. Use a non-prefixed form when one exists, but for properties that still need a prefix, this works.

## Custom Properties and `var()` Cascade

Because custom properties inherit through the DOM, you can set them on a parent and read them deep in the tree.

```svelte
<!-- App.svelte -->
<script>
  let theme = $state('light');
</script>

<div
  style:--bg={theme === 'light' ? '#fff' : '#111'}
  style:--text={theme === 'light' ? '#111' : '#eee'}
>
  <Card>
    <DeepChild />
  </Card>
</div>

<style>
  div {
    background: var(--bg);
    color: var(--text);
  }
</style>
```

The custom properties cascade to `<Card>` and `<DeepChild>` automatically; nothing extra is needed.

## Common Mistakes

### Forgetting the unit on a number

```svelte
<!-- WRONG: results in width: 10 (no unit, invalid for width) -->
<div style:width={10}></div>

<!-- RIGHT -->
<div style:width="{10}px"></div>
```

### Using `style:--color` without `--`

```svelte
<!-- WRONG: not a custom property name -->
<div style:color-scheme="dark">...</div> <!-- this is fine, but -->
<div style:color-scheme>...</div>      <!-- the variable name "color-scheme" is fine, this is regular CSS property -->
```

`style:--` is required **only** for custom properties (variables). For regular CSS properties, write them without `--`.

### Confusing `style:` with `class:`

```svelte
<!-- style: sets CSS properties -->
<div style:color="red">

<!-- class: sets class names (deprecated in 5.16+, use class={}) -->
<div class:cool={isCool}>
```

### Treating the value as a CSS object

`style:` takes a single value (string or expression returning a string), not an object.

```svelte
<!-- WRONG -->
<div style:{{ color: 'red', width: '10px' }} />

<!-- RIGHT: use multiple style: directives or a style attribute -->
<div style:color="red" style:width="10px" />
<div style="color: red; width: 10px" />
```

## See also

- [references/style-directive.md](../references/style-directive.md) — full reference on `style:`, `|important`, and precedence
- [references/css-custom-properties.md](../references/css-custom-properties.md) — custom properties patterns
