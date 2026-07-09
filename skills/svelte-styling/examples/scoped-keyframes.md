# Scoped Keyframes Examples

`@keyframes` declared inside a component's `<style>` block is automatically **scoped** to that component. The animation name is renamed (similar to the class hash), so it cannot collide with keyframes in other components — and the `animation:` shorthand inside the component is rewritten to match.

This file shows every keyframe scoping pattern in Svelte 5.

## Table of Contents

- [Basic Scoped Keyframe](#basic-scoped-keyframe)
- [Compiled: What Happens to the Name](#compiled-what-happens-to-the-name)
- [Scoped Keyframe Used in an `animation` Shorthand](#scoped-keyframe-used-in-an-animation-shorthand)
- [Global Keyframe with `-global-` Prefix](#global-keyframe-with--global--prefix)
- [Reusing a Global Keyframe Across Components](#reusing-a-global-keyframe-across-components)
- [Mixing Scoped and Global in One Component](#mixing-scoped-and-global-in-one-component)
- [Keyframes Triggered Programmatically (style.animation)](#keyframes-triggered-programmatically-styleanimation)
- [CSS Animation with Scoped + Global Together](#css-animation-with-scoped--global-together)
- [Reduced Motion + Keyframes](#reduced-motion--keyframes)

## Basic Scoped Keyframe

Declare the keyframe in the same `<style>` block where it's used. The name is auto-scoped.

```svelte
<script>
  let playing = $state(false);
</script>

<style>
  @keyframes bounce {
    0%, 100% { transform: translateY(0); }
    50%      { transform: translateY(-20px); }
  }

  .box {
    width: 80px;
    height: 80px;
    background: #ff3e00;
    border-radius: 8px;
    animation: bounce 1s ease-in-out infinite;
  }

  .paused { animation-play-state: paused; }
</style>

<div class="box {playing ? '' : 'paused'}"></div>
<button onclick={() => playing = !playing}>
  {playing ? 'Pause' : 'Play'}
</button>
```

The compiled CSS effectively becomes:

```css
@keyframes bounce-svelte-abc123 { /* ... */ }
.box.svelte-abc123 { animation: bounce-svelte-abc123 1s ease-in-out infinite; }
```

No need to rename the keyframe yourself — the compiler does it, and the `animation` reference is rewritten to match.

## Compiled: What Happens to the Name

Svelte hashes the keyframe using the same scheme as the scope class. The actual name in the emitted CSS is something like `bounce-svelte-abc123` (form may vary across versions). The important part: **the hash is consistent within the component**, so any `animation: bounce ...` inside the same component is rewritten to `animation: bounce-svelte-abc123 ...`.

```svelte
<style>
  /* Source: */
  @keyframes wiggle { 0% { transform: rotate(0); } 100% { transform: rotate(5deg); } }
  .thing { animation: wiggle 0.3s; }
</style>
```

```css
/* Compiled: */
@keyframes wiggle-svelte-xyz { 0% { transform: rotate(0); } 100% { transform: rotate(5deg); } }
.thing.svelte-xyz { animation: wiggle-svelte-xyz 0.3s; }
```

If two components both define `@keyframes wiggle`, the compiled output uses two different hashes — no clash.

## Scoped Keyframe Used in an `animation` Shorthand

The compiler only rewrites `animation` (and the longhand `animation-name`) declarations **inside the same component's `<style>`**. If you write the animation inline via the `style` attribute, the unscoped name won't match.

```svelte
<style>
  @keyframes wiggle { /* ... */ }
</style>

<!-- WRONG: 'wiggle' is scoped, so style="animation: wiggle ..." won't find it -->
<div style="animation: wiggle 0.3s">Wiggle</div>

<!-- CORRECT: use a scoped class -->
<style>
  .wiggle-me { animation: wiggle 0.3s; }
</style>
<div class="wiggle-me">Wiggle (works)</div>
```

Or use a `-global-` keyframe if you need to drive it from an inline style.

## Global Keyframe with `-global-` Prefix

Prepend the keyframe name with `-global-`. The compiler **strips** the prefix and emits the keyframe under its bare name, available to the entire app.

```svelte
<style>
  @keyframes -global-fade-in {
    from { opacity: 0; transform: translateY(8px); }
    to   { opacity: 1; transform: translateY(0); }
  }
</style>

<div class="fade-target">Fades in</div>

<style>
  .fade-target { animation: fade-in 0.4s ease-out; }
</style>
```

Now `fade-in` is registered globally. Other components can use `animation: fade-in ...` or set `element.style.animation = 'fade-in 0.4s'` and it just works.

## Reusing a Global Keyframe Across Components

Define the keyframe once with `-global-` in any component, then reference it from anywhere.

```svelte
<!-- animations.svelte (lives at app shell) -->
<style>
  @keyframes -global-shimmer {
    0%   { background-position: -200% 0; }
    100% { background-position: 200% 0; }
  }
</style>
```

```svelte
<!-- Skeleton.svelte (consumer) -->
<style>
  .skeleton {
    background: linear-gradient(90deg, #eee 25%, #f5f5f5 50%, #eee 75%);
    background-size: 200% 100%;
    animation: shimmer 1.4s linear infinite;
  }
</style>

<div class="skeleton">Loading…</div>
```

Because Svelte emits `shimmer` as a global `@keyframes` rule (no hash), the consumer's `animation: shimmer ...` finds it.

## Mixing Scoped and Global in One Component

You can have both scoped and global keyframes side by side. Use the prefix consistently.

```svelte
<style>
  /* Global — reusable across the app */
  @keyframes -global-pulse {
    0%, 100% { opacity: 1; }
    50%      { opacity: 0.5; }
  }

  /* Scoped — only this component */
  @keyframes wiggle {
    0%, 100% { transform: rotate(0); }
    50%      { transform: rotate(3deg); }
  }

  .pulse  { animation: pulse 1.2s ease-in-out infinite; }
  .wiggle { animation: wiggle 0.4s ease-in-out; }
</style>

<div class="pulse">Pulsing</div>
<div class="wiggle">Wiggling</div>
```

`pulse` is emitted globally; `wiggle` is emitted scoped to this component only.

## Keyframes Triggered Programmatically (style.animation)

If you need to set the animation from JavaScript (e.g. on user click), use a **global** keyframe — otherwise the bare name in `element.style.animation` won't resolve to the scoped one.

```svelte
<script>
  let el;
  function play() {
    el.style.animation = 'fade-in 0.4s ease-out';
  }
</script>

<style>
  @keyframes -global-fade-in {
    from { opacity: 0; }
    to   { opacity: 1; }
  }
</style>

<div bind:this={el}>Reveal me</div>
<button onclick={play}>Play</button>
```

For one-off component animations triggered from JS, prefer toggling a class that owns the `animation` rule. That way the `animation-name` is written in the `<style>` block and the compiler rewrites it for you.

## CSS Animation with Scoped + Global Together

You can reference a global keyframe from a scoped rule, and a scoped keyframe from inside the same component only.

```svelte
<style>
  @keyframes -global-spin {
    to { transform: rotate(360deg); }
  }

  @keyframes pulse-scale {
    0%, 100% { transform: scale(1); }
    50%      { transform: scale(1.05); }
  }

  .spinner { animation: spin 1s linear infinite; }        /* global */
  .pulse-card { animation: pulse-scale 0.8s ease-in-out; } /* scoped */
</style>
```

`spin` is shared app-wide; `pulse-scale` is private to this component.

## Reduced Motion + Keyframes

Honour the user's `prefers-reduced-motion` setting by disabling animations while keeping the keyframes declared.

```svelte
<style>
  @keyframes -global-slide-in {
    from { transform: translateX(-20px); opacity: 0; }
    to   { transform: translateX(0);     opacity: 1; }
  }

  .reveal {
    animation: slide-in 0.4s ease-out;
  }

  @media (prefers-reduced-motion: reduce) {
    .reveal { animation: none; }
  }
</style>

<div class="reveal">Hello</div>
```

This pattern is independent of whether the keyframe is scoped or global — both respect the media query the same way.

## See also

- [references/scoped-keyframes.md](../references/scoped-keyframes.md) — full reference on scoping, naming, and the `-global-` rule
- [references/global-reference.md](../references/global-reference.md) — `:global(...)` for selectors and keyframes
- [references/specificity.md](../references/specificity.md) — when scoped + global rules interact
