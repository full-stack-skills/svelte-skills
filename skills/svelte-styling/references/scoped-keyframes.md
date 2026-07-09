# Scoped Keyframes Reference

A complete reference for `@keyframes` scoping in Svelte 5.

## The Rule

`@keyframes` declared in a component's `<style>` block is **automatically scoped** to that component. The compiler renames the keyframe (so it can't collide with keyframes in other components) and rewrites any `animation` / `animation-name` references inside the same component to match.

```svelte
<style>
  @keyframes wiggle { /* ... */ }
  .thing { animation: wiggle 0.3s; }
</style>
```

Compiles to (conceptually):

```css
@keyframes wiggle-svelte-abc123 { /* ... */ }
.thing.svelte-abc123 { animation: wiggle-svelte-abc123 0.3s; }
```

The exact form of the suffix varies across Svelte versions, but the *contract* is: within the same component, `wiggle` and `animation: wiggle` always refer to the same name; across components, the names never collide.

## The `-global-` Prefix

To make a keyframe **not** scoped — i.e. available to the entire app — prepend the keyframe name with `-global-`. The prefix is stripped during compilation.

```svelte
<style>
  @keyframes -global-fade-in {
    from { opacity: 0; }
    to   { opacity: 1; }
  }
</style>
```

Compiles to:

```css
@keyframes fade-in { from { opacity: 0; } to { opacity: 1; } }
```

Now any other component (or even inline JS) can use `animation: fade-in ...` or set `element.style.animation = 'fade-in 0.4s'`, and it will resolve to the global rule.

### Naming convention

Use the `-global-` prefix exactly — no spaces, no other punctuation. The compiler looks for this prefix as a literal token.

```svelte
<style>
  @keyframes -global-shimmer { /* OK */ }
  @keyframes -global-  spin  { /* WRONG: extra space, won't be recognized as a global keyframe */ }
  @keyframes shimmer { /* scoped, not global */ }
</style>
```

## When the Compiler Rewrites `animation`

The compiler rewrites the `animation` and `animation-name` properties **only inside the same component's scoped CSS**. It does **not** rewrite:

- `style="animation: name ..."` attributes in your markup.
- Inline JS that sets `element.style.animation = 'name ...'`.
- Selectors in nested `<style>` tags.
- Rules in `:global { ... }` blocks (those are emitted as-is, so they need the global name to start with).

For these cases, use a `-global-` keyframe so the name you write is the name the browser sees.

## How the Rewriter Picks the Name

For each `@keyframes` declaration in a scoped `<style>`:

1. Compute the keyframe's new name as `<original>-svelte-<hash>` (form may vary by Svelte version).
2. In every `animation` / `animation-name` declaration in the same `<style>` block, replace the original name with the new one.
3. Emit the `@keyframes` itself with the new name.

The hash is the same as the component's scope class hash, so the keyframe name and the scope class name are consistent within a component.

## Sharing a Keyframe Across Components

There are three approaches, in order of preference:

### 1. Define in a top-level shell component with `-global-`

```svelte
<!-- src/lib/animations.svelte -->
<style>
  @keyframes -global-fade-in { from { opacity: 0; } to { opacity: 1; } }
  @keyframes -global-shimmer { 0% { background-position: -200% 0; } 100% { background-position: 200% 0; } }
</style>
```

Import this component once at the app root (or render it once with no content). The keyframes are registered globally; everyone can reference them by bare name.

### 2. Put them in a global stylesheet

```css
/* src/app.css */
@keyframes fade-in { from { opacity: 0; } to { opacity: 1; } }
```

```svelte
<!-- App.svelte -->
<script>
  import './app.css';
</script>
```

This is identical in effect to approach 1 but separates CSS from components.

### 3. Pass the keyframe name down via a custom property (rare)

```svelte
<style>
  .thing { animation: var(--anim-name) 0.3s; }
</style>
```

```svelte
<Thing --anim-name="fade-in" />
```

Useful when the consumer doesn't know the animation name in advance.

## Multiple `@keyframes` in One Component

All are scoped independently. The compiler renames each one.

```svelte
<style>
  @keyframes spin { to { transform: rotate(360deg); } }
  @keyframes pulse { 50% { opacity: 0.5; } }

  .spinner { animation: spin 1s linear infinite; }
  .pulse   { animation: pulse 1.2s ease-in-out infinite; }
</style>
```

Compiled:

```css
@keyframes spin-svelte-abc123 { to { transform: rotate(360deg); } }
@keyframes pulse-svelte-abc123 { 50% { opacity: 0.5; } }
.spinner.svelte-abc123 { animation: spin-svelte-abc123 1s linear infinite; }
.pulse.svelte-abc123   { animation: pulse-svelte-abc123 1.2s ease-in-out infinite; }
```

## Animations and `:global`

If a `:global { ... }` block contains an `animation` rule, the rule is emitted as global CSS. The compiler **does not rewrite the keyframe name in `:global` blocks**. Therefore the keyframe must be defined with `-global-` or in a global stylesheet.

```svelte
<style>
  @keyframes -global-fade-in { from { opacity: 0; } to { opacity: 1; } }

  :global {
    /* This is emitted as global CSS. The keyframe name 'fade-in'
       is preserved by the compiler. */
    .global-fade { animation: fade-in 0.3s ease-out; }
  }
</style>
```

If the keyframe were scoped, this would emit `animation: fade-in-svelte-abc123 0.3s` in global CSS, which would not match the scoped `@keyframes` name (which only exists in the component's own scope).

## `@keyframes` and Nested `<style>`

Nested `<style>` tags are inserted into the DOM as-is — no scoping, no rewriting.

```svelte
{#if dark}
  <style>
    @keyframes pulse { 50% { opacity: 0.5; } }
    body { animation: pulse 2s infinite; }
  </style>
{/if}
```

Because there is no scope hash, the `@keyframes` name is emitted verbatim. To avoid clashing with another component's `@keyframes pulse`, give the nested style's animations unique names or use a `-global-` prefix in the top-level style and reference that name.

## Interaction with `prefers-reduced-motion`

Honour the user's motion preference regardless of how the keyframe is scoped:

```svelte
<style>
  @keyframes -global-slide-in { from { transform: translateX(-20px); opacity: 0; } to { transform: translateX(0); opacity: 1; } }

  .reveal { animation: slide-in 0.4s ease-out; }

  @media (prefers-reduced-motion: reduce) {
    .reveal { animation: none; }
  }
</style>
```

This works for both scoped and global keyframes — the media query doesn't care where the `@keyframes` was defined.

## Summary

| Pattern | Compiler Behavior | Use it when |
|---------|-------------------|-------------|
| `@keyframes foo { ... }` (no prefix) | Scoped, name suffixed with hash | Animation used only inside the component |
| `@keyframes -global-foo { ... }` | Stripped of prefix, name emitted as-is | Animation shared across the app |
| `animation: foo ...` in same scoped `<style>` | Rewritten to match scoped name | Always, when keyframe is scoped |
| `animation: foo ...` outside a scoped `<style>` (e.g. in `:global`, in inline `style=`, in nested `<style>`) | NOT rewritten — use a global name | Animation triggered from outside the component |
| Multiple `@keyframes` in one component | Each independently scoped | Per-component animations |
| `prefers-reduced-motion` | Independent of scoping | Always, for accessibility |

## See also

- [examples/scoped-keyframes.md](../examples/scoped-keyframes.md) — practical examples for every pattern
- [examples/global-patterns.md](../examples/global-patterns.md) — `:global(...)` selector rules
- [references/global-reference.md](./global-reference.md) — full reference for `:global(...)` forms
- [references/specificity.md](./specificity.md) — how scoped and global rules interact in the cascade
