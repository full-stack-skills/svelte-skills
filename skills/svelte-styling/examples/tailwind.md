# Tailwind CSS Integration Examples

This file covers Tailwind CSS integration patterns in Svelte 5.

## Table of Contents

- [Basic Tailwind Setup](#basic-tailwind-setup)
- [Using Arbitrary Values](#using-arbitrary-values)
- [Dynamic Tailwind Classes with $state](#dynamic-tailwind-classes-with-state)
- [Dark Mode with Tailwind](#dark-mode-with-tailwind)
- [Class Composition with Child Components](#class-composition-with-child-components)
- [Card Component with Tailwind](#card-component-with-tailwind)
- [Form with Tailwind Validation States](#form-with-tailwind-validation-states)
- [Responsive Tailwind with Svelte State](#responsive-tailwind-with-svelte-state)
- [Arbitrary Values Deep Dive](#arbitrary-values-deep-dive)
- [Dynamic Theming with Custom Properties](#dynamic-theming-with-custom-properties)
- [Plugin Components (Forms, Buttons)](#plugin-components-forms-buttons)
- [Safelist for Dynamic Class Generation](#safelist-for-dynamic-class-generation)
- [Avoiding Class Drift with `clsx`/Class Arrays](#avoiding-class-drift-with-clsxclass-arrays)

## Basic Tailwind Setup

Configure Tailwind in your Svelte project:

```js
// svelte.config.js
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

export default {
  preprocess: vitePreprocess(),
};
```

```js
// tailwind.config.js
export default {
  content: ['./src/**/*.{html,js,svelte,ts}'],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

```css
/* app.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

```svelte
<!-- App.svelte -->
<script>
  import '../app.css';
</script>

<slot />
```

---

## Using Arbitrary Values

Tailwind's arbitrary values work directly in class attributes.

```svelte
<!-- Arbitrary values in square brackets -->
<div class="w-[calc(100%-2rem)] p-[1.5rem]">
  Arbitrary width and padding
</div>

<!-- Arbitrary colors -->
<button class="bg-[#1da1f2] text-[#ffffff]">
  Custom color button
</button>

<!-- Arbitrary Tailwind values -->
<div class="text-[clamp(1rem,2vw,2rem)]">
  Responsive text with clamp
</div>
```

---

## Dynamic Tailwind Classes with $state

Use Svelte 5's `$state` rune for dynamic Tailwind classes.

```svelte
<script>
  let isDark = $state(false);
  let size = $state('md'); // 'sm' | 'md' | 'lg'
  let isDisabled = $state(false);
</script>

<style>
  .btn {
    @apply transition-all duration-200;
  }
</style>

<!-- Dynamic dark mode -->
<div class={isDark ? 'bg-gray-900 text-white' : 'bg-white text-gray-900'}>
  <p>Current mode: {isDark ? 'Dark' : 'Light'}</p>
</div>

<!-- Dynamic size -->
<button
  class={[
    'btn px-4 py-2 rounded font-medium',
    size === 'sm' && 'text-sm',
    size === 'md' && 'text-base',
    size === 'lg' && 'text-lg px-6 py-3',
    isDisabled && 'opacity-50 cursor-not-allowed',
    !isDisabled && 'hover:bg-blue-600'
  ]}
  disabled={isDisabled}
>
  Button ({size})
</button>
```

---

## Dark Mode with Tailwind

### Tailwind Dark Mode (class strategy)

```svelte
<script>
  let isDark = $state(false);

  function toggleDark() {
    isDark = !isDark;
    if (isDark) {
      document.documentElement.classList.add('dark');
    } else {
      document.documentElement.classList.remove('dark');
    }
  }
</script>

<style>
  @tailwind base;
  @tailwind components;
  @tailwind utilities;
</style>

<!-- Using Tailwind's dark: variant -->
<div class="min-h-screen bg-white dark:bg-gray-900 transition-colors">
  <nav class="p-4 flex justify-between items-center bg-gray-100 dark:bg-gray-800">
    <span class="text-gray-900 dark:text-gray-100">Logo</span>
    <button
      onclick={toggleDark}
      class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
    >
      Toggle Dark Mode
    </button>
  </nav>

  <main class="p-8">
    <h1 class="text-3xl font-bold text-gray-900 dark:text-white mb-4">
      Tailwind Dark Mode
    </h1>
    <p class="text-gray-600 dark:text-gray-300">
      This text adapts to dark mode automatically.
    </p>
  </main>
</div>
```

---

## Class Composition with Child Components

Pass Tailwind classes to child components.

```svelte
<!-- Button.svelte -->
<script>
  let { class: cls = '', variant = 'primary', ...props } = $props();
</script>

<style>
  @tailwind base;
  @tailwind components;
  @tailwind utilities;
</style>

<button
  class={[
    'px-4 py-2 rounded font-medium transition-colors',
    variant === 'primary' && 'bg-blue-500 text-white hover:bg-blue-600',
    variant === 'secondary' && 'bg-gray-200 text-gray-800 hover:bg-gray-300',
    variant === 'ghost' && 'bg-transparent hover:bg-gray-100',
    cls
  ]}
  {...props}
>
  <slot />
</button>
```

```svelte
<!-- Usage -->
<Button variant="primary" class="ml-2">
  Primary with extra margin
</Button>

<Button variant="ghost" class="text-red-500 hover:text-red-600">
  Ghost with custom color
</Button>
```

---

## Card Component with Tailwind

```svelte
<!-- Card.svelte -->
<script>
  let {
    title,
    description,
    image,
    class: cls = '',
    ...props
  } = $props();
</script>

<style>
  @tailwind base;
  @tailwind components;
  @tailwind utilities;

  @layer components {
    .card-shadow {
      @apply shadow-lg hover:shadow-xl transition-shadow;
    }
  }
</style>

<div
  class={[
    'bg-white rounded-xl overflow-hidden card-shadow',
    'border border-gray-200',
    cls
  ]}
  {...props}
>
  {#if image}
    <img src={image} alt={title} class="w-full h-48 object-cover" />
  {/if}

  <div class="p-6">
    <h3 class="text-xl font-semibold text-gray-900 mb-2">{title}</h3>
    <p class="text-gray-600">{description}</p>
  </div>
</div>
```

```svelte
<!-- Usage -->
<Card
  title="Svelte 5"
  description="The official Svelte plugin for Tailwind"
  image="https://example.com/svelte.jpg"
  class="max-w-sm mx-auto mt-8"
/>
```

---

## Form with Tailwind Validation States

```svelte
<script>
  let email = $state('');
  let isValid = $state(false);
  let isTouched = $state(false);

  $effect(() => {
    isValid = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  });
</script>

<style>
  @tailwind base;
  @tailwind components;
  @tailwind utilities;
</style>

<form class="max-w-md mx-auto p-6 space-y-4">
  <div>
    <label class="block text-sm font-medium text-gray-700 mb-1">
      Email
    </label>
    <input
      type="email"
      bind:value={email}
      onblur={() => isTouched = true}
      class={[
        'w-full px-4 py-2 border rounded-lg transition-colors',
        'focus:outline-none focus:ring-2 focus:ring-blue-500',
        isTouched && !isValid && 'border-red-500 focus:ring-red-500',
        isTouched && isValid && 'border-green-500 focus:ring-green-500',
        !isTouched && 'border-gray-300'
      ]}
      placeholder="you@example.com"
    />
    {#if isTouched && !isValid}
      <p class="mt-1 text-sm text-red-500">Please enter a valid email</p>
    {/if}
  </div>

  <button
    type="submit"
    disabled={!isValid}
    class={[
      'w-full py-2 px-4 rounded-lg font-medium transition-colors',
      isValid
        ? 'bg-blue-500 text-white hover:bg-blue-600'
        : 'bg-gray-300 text-gray-500 cursor-not-allowed'
    ]}
  >
    Submit
  </button>
</form>
```

---

## Responsive Tailwind with Svelte State

```svelte
<script>
  let isMenuOpen = $state(false);
  let activeTab = $state('home');

  const tabs = ['home', 'about', 'contact'];
</script>

<style>
  @tailwind base;
  @tailwind components;
  @tailwind utilities;
</style>

<div class="min-h-screen bg-gray-50">
  <!-- Mobile menu button -->
  <button
    class="md:hidden p-4"
    onclick={() => isMenuOpen = !isMenuOpen}
  >
    <svg class="w-6 h-6">{isMenuOpen ? 'X' : 'Menu'}</svg>
  </button>

  <!-- Desktop navigation -->
  <nav class="hidden md:flex gap-4 p-4 bg-white shadow">
    {#each tabs as tab}
      <button
        onclick={() => activeTab = tab}
        class={[
          'px-4 py-2 rounded-lg transition-colors',
          activeTab === tab
            ? 'bg-blue-500 text-white'
            : 'text-gray-600 hover:bg-gray-100'
        ]}
      >
        {tab}
      </button>
    {/each}
  </nav>

  <!-- Mobile navigation -->
  {#if isMenuOpen}
    <nav class="md:hidden p-4 bg-white shadow-lg">
      {#each tabs as tab}
        <button
          onclick={() => { activeTab = tab; isMenuOpen = false; }}
          class={[
            'block w-full text-left px-4 py-3 rounded-lg transition-colors',
            activeTab === tab
              ? 'bg-blue-500 text-white'
              : 'text-gray-600 hover:bg-gray-100'
          ]}
        >
          {tab}
        </button>
      {/each}
    </nav>
  {/if}

  <!-- Content -->
  <main class="p-8">
    <h1 class="text-2xl md:text-4xl font-bold text-gray-900">
      Responsive Tab: {activeTab}
    </h1>
  </main>
</div>
```

---

## Arbitrary Values Deep Dive

Tailwind lets you escape the design system with square-bracket arbitrary values. In Svelte 5, the class attribute can be an object/array, so you can compose arbitrary values alongside conditionals.

### Lengths and Sizing

```svelte
<div class="w-[640px] h-[clamp(200px,40vh,400px)]">
  Fixed width, responsive height via clamp
</div>

<div class="min-h-[100dvh]">
  Use dvh (dynamic viewport) for mobile-safe full screen
</div>
```

### Colors

```svelte
<!-- Hex, rgb, hsl -->
<div class="bg-[#1da1f2]">Brand color</div>
<div class="text-[rgb(255_100_50)]">RGB color</div>
<div class="border-[hsl(220,14%,30%)]">HSL border</div>

<!-- CSS variables -->
<div class="bg-[var(--brand)]">From custom property</div>
```

### Typography

```svelte
<p class="text-[clamp(1rem,1.2vw,1.5rem)] leading-[1.4]">
  Fluid type using clamp() and arbitrary line height
</p>

<p class="tracking-[-0.02em]">
  Tighter-than-default tracking
</p>
```

### Grids and Flex

```svelte
<div class="grid grid-cols-[1fr_2fr_1fr] gap-[1.25rem]">
  <div>Left</div>
  <div>Wider middle</div>
  <div>Right</div>
</div>

<div class="flex [&>*:nth-child(3)]:order-last">
  <!-- Arbitrary variant targeting the 3rd child -->
  <div>1</div>
  <div>2</div>
  <div>3 (last)</div>
</div>
```

### Arbitrary Variants

Bracket syntax also works in *variant* position — `class-[is-active]:bg-blue-500` matches when the element has the class `is-active`. Useful for ad-hoc conditional styling without writing a custom variant.

```svelte
<div class="[&.is-active]:bg-blue-500 [&.is-active]:text-white">
  <span class={isActive ? 'is-active' : ''}>Status</span>
</div>
```

---

## Dynamic Theming with Custom Properties

Combining Tailwind with CSS custom properties gives a fully theme-able design without rebuilding utility classes.

```js
// app.css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --color-bg: 0 0% 100%;
    --color-fg: 222 47% 11%;
    --color-accent: 14 100% 50%;
  }

  .dark {
    --color-bg: 222 47% 6%;
    --color-fg: 0 0% 95%;
    --color-accent: 14 100% 60%;
  }
}
```

```svelte
<!-- ThemedCard.svelte -->
<script>
  let { class: cls = '' } = $props();
</script>

<div
  class={[
    'rounded-xl p-6 shadow-md',
    'bg-[hsl(var(--color-bg))] text-[hsl(var(--color-fg))]',
    'border border-[hsl(var(--color-fg)/0.1)]',
    cls
  ]}
>
  <slot />
</div>
```

```svelte
<!-- App.svelte -->
<script>
  import { onMount } from 'svelte';
  let isDark = $state(false);

  onMount(() => {
    isDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    document.documentElement.classList.toggle('dark', isDark);
  });

  function toggle() {
    isDark = !isDark;
    document.documentElement.classList.toggle('dark', isDark);
  }
</script>

<button onclick={toggle} class="px-3 py-1 rounded border">
  {isDark ? 'Light' : 'Dark'}
</button>

<ThemedCard>Adapts to the current theme automatically.</ThemedCard>
```

### Why store as `H S L` channels

Storing the variables as space-separated `H S L` channels (no `hsl()` wrapper) lets you compose opacity with Tailwind's slash syntax: `bg-[hsl(var(--color-bg)/0.5)]`.

---

## Plugin Components (Forms, Buttons)

Build a small library of form primitives that compose with Tailwind.

```svelte
<!-- TextField.svelte -->
<script lang="ts">
  import type { ClassValue } from 'svelte/elements';
  let {
    value = $bindable(''),
    label,
    error = '',
    class: cls = '',
    ...props
  }: {
    value?: string;
    label?: string;
    error?: string;
    class?: ClassValue;
    [key: string]: unknown;
  } = $props();
</script>

<label class="flex flex-col gap-1 text-sm">
  {#if label}
    <span class="font-medium text-gray-700">{label}</span>
  {/if}

  <input
    bind:value
    class={[
      'px-3 py-2 rounded-md border transition-colors',
      'focus:outline-none focus:ring-2 focus:ring-blue-500',
      error
        ? 'border-red-400 focus:ring-red-400'
        : 'border-gray-300 focus:border-blue-500',
      cls
    ]}
    {...props}
  />

  {#if error}
    <span class="text-red-500 text-xs">{error}</span>
  {/if}
</label>
```

```svelte
<!-- Usage -->
<TextField
  label="Email"
  type="email"
  bind:value={email}
  error={touched && !valid ? 'Invalid email' : ''}
  class="max-w-sm"
/>
```

---

## Safelist for Dynamic Class Generation

Tailwind's JIT scans source files for class names. If you build class names dynamically (concatenation, lookup maps), the classes may not appear in the output.

```js
// tailwind.config.js
export default {
  content: ['./src/**/*.{html,js,svelte,ts}'],
  safelist: [
    // For known variants
    'bg-red-500', 'bg-yellow-500', 'bg-green-500',
    'text-red-500', 'text-yellow-500', 'text-green-500',
  ],
};
```

Or, better: keep dynamic class names **as full string literals somewhere in your code**, so the scanner can find them.

```svelte
<!-- App.svelte -->
<script>
  const STATUS_CLASSES = {
    ok:    'bg-green-500 text-white',
    warn:  'bg-yellow-500 text-black',
    error: 'bg-red-500 text-white',
  };

  let status = $state('ok');
</script>

<!-- All three class names appear in the file, so the JIT includes them -->
<span class={STATUS_CLASSES[status]}>{status}</span>
```

### `bg-${color}-500` is a footgun

```svelte
<!-- This pattern is fragile: the JIT can't see "bg-red-500" in the source -->
<button class={`bg-${color}-500`}>
```

The literal `bg-`, the dynamic `color`, and `-500` are *strings* — the JIT cannot reconstruct the full class. Always prefer lookup maps or safelisting.

---

## Avoiding Class Drift with `clsx`/Class Arrays

The `class` attribute in Svelte 5.16+ is itself a small clsx implementation: objects and arrays are flattened, falsy values are dropped. Use this to compose Tailwind without worrying about whitespace, `undefined`, or trailing spaces.

```svelte
<script>
  let { isPrimary, isDisabled, isLoading, class: cls = '' } = $props();
</script>

<button
  class={[
    'px-4 py-2 rounded font-medium transition-colors',
    isPrimary
      ? 'bg-blue-500 text-white hover:bg-blue-600'
      : 'bg-white text-gray-800 border border-gray-300 hover:bg-gray-50',
    isDisabled && 'opacity-50 cursor-not-allowed',
    isLoading && 'animate-pulse',
    cls
  ]}
  disabled={isDisabled || isLoading}
>
  <slot />
</button>
```

The compiled class string is built from the truthy entries — no `undefined`, no `false` text sneaking in.

### Use `twMerge` for real conflict resolution

If you want later utilities to win over earlier ones (e.g. user-supplied `class="p-4"` overriding your base `class="p-2"`), import `tailwind-merge` and use it explicitly.

```svelte
<script>
  import { twMerge } from 'tailwind-merge';

  let { class: cls = '' } = $props();
</script>

<button class={twMerge('p-2 bg-blue-500', cls)}>
  <slot />
</button>
```

`twMerge` understands Tailwind's class hierarchy, so `p-4` will replace `p-2` in the final string, while `bg-red-500` will not clobber `bg-blue-500` (because of the `tailwind-merge` color conflict rules).

---

## See also

- [examples/scoped-styles.md](./scoped-styles.md) — base scoping rules for `<style>` blocks
- [examples/style-directive.md](./style-directive.md) — `style:--var` for passing custom properties to children
- [references/css-custom-properties.md](../references/css-custom-properties.md) — custom properties in depth

