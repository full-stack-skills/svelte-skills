# Tailwind CSS Integration Examples

This file covers Tailwind CSS integration patterns in Svelte 5.

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
