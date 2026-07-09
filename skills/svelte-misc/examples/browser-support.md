# Browser Support Examples

Svelte 5 targets the **Baseline 2020** browser set. This file shows how to detect feature usage, polyfill, configure bundlers, and handle the few features that need newer browsers.

---

## 1. Minimum Browser Versions (Baseline 2020)

| Browser | Minimum version |
| - | - |
| Chrome / Edge | 87 |
| Firefox | 83 |
| Safari | 14 |
| Opera | 73 |
| Opera (Android) | 62 |
| Samsung Internet | 14.0 |
| Android WebView | 87 |
| Internet Explorer | not supported |

If you need to support older browsers, plan on polyfills.

---

## 2. Feature-Specific Exceptions

A few Svelte features need newer browsers. The compiler warns when you use them; gate them with feature detection or polyfills.

| Feature | Chrome/Edge | Firefox | Safari |
| - | - | - | - |
| `$state.snapshot` | 98 | 94 | 15.4 |
| `bind:devicePixelContentBoxSize` | — | 93 | not supported |
| `flip` from `svelte/animate` | — | 126 | — |

Example: gate `flip` on Firefox versions that support it:

```svelte
<script>
  import { flip } from 'svelte/animate';
  import { quintOut } from 'svelte/easing';

  let supportsFlip = typeof window !== 'undefined' &&
    CSS.supports('animation-timeline', 'view()'); // proxy check
</script>

{#if supportsFlip}
  <ul>
    {#each items as item (item.id)}
      <li animate:flip={{ duration: 300, easing: quintOut }}>{item.text}</li>
    {/each}
  </ul>
{/if}
```

---

## 3. Polyfill Web Components for Older Browsers

Custom elements require a polyfill on browsers without native support (older Edge, pre-2020 Safari). The `@webcomponents/webcomponentsjs` package provides them.

Install:

```bash
npm install @webcomponents/webcomponentsjs
```

Conditionally load in your app entry:

```html
<!-- src/app.html or index.html -->
<script>
  if (!('customElements' in window)) {
    // @ts-ignore
    import('@webcomponents/webcomponentsjs/custom-elements.min.js');
  }
</script>
```

With Vite, use dynamic import + feature detection:

```ts
// src/main.ts
async function loadPolyfills() {
  if (!window.customElements) {
    await import('@webcomponents/webcomponentsjs/custom-elements.min.js');
  }
  if (!window.shadowRoot) {
    // Some browsers have customElements but no shadow DOM
    // (e.g. older Edge with the Shady DOM polyfill)
  }
}

loadPolyfills().then(() => {
  // import your Svelte components after polyfills load
  import('./App.svelte');
});
```

---

## 4. Adjusting `target` in `tsconfig.json` / Bundler

Svelte's runtime is shipped as ES2015+. Make sure your build target matches the browsers you support.

For Vite (browser target):

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import { svelte } from '@sveltejs/vite-plugin-svelte';

export default defineConfig({
  plugins: [svelte()],
  build: {
    target: 'es2020' // or 'es2015' for Baseline 2020
  }
});
```

For TypeScript:

```json
{
  "compilerOptions": {
    "target": "ES2015",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "moduleResolution": "Bundler"
  }
}
```

> Setting `target` to `ES2015` or higher is required to keep classes as classes in compiled output.

---

## 5. SSR — Guarding Browser-Only Code

`window`, `document`, and other DOM APIs don't exist on the server. Use the `browser` export from `$app/environment` (SvelteKit) or `typeof window`:

```svelte
<!-- In SvelteKit -->
<script>
  import { browser } from '$app/environment';

  if (browser) {
    // Browser-only setup
    window.addEventListener('resize', handleResize);
  }
</script>
```

Without SvelteKit, prefer `<svelte:window>` for listeners (Svelte handles SSR/CSR transparently):

```svelte
<script>
  function handleResize() { /* ... */ }
</script>

<svelte:window onresize={handleResize} />
```

> Effects never run on the server, so `$effect` is always safe for browser-only code inside the body of the effect.

---

## 6. Detecting `$state.snapshot` Support

`$state.snapshot` returns a plain (non-proxied) copy of state. It uses `structuredClone` under the hood — needs Chrome 98+, Firefox 94+, Safari 15.4+.

Polyfill fallback:

```ts
function snapshot<T>(state: T): T {
  if (typeof structuredClone === 'function') {
    return structuredClone(state);
  }
  return JSON.parse(JSON.stringify(state));
}
```

Or check before using:

```svelte
<script>
  const supportsSnapshot = typeof structuredClone === 'function';
  let list = $state([1, 2, 3]);
  const plain = supportsSnapshot ? $state.snapshot(list) : [...list];
</script>
```

---

## 7. Testing Older Browsers

Use Playwright's `browsers` config to actually run your app in the minimum supported browsers:

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    { name: 'chrome-min', use: { ...devices['Desktop Chrome'], channel: 'chrome', browserVersion: '87' } },
    { name: 'firefox-min', use: { ...devices['Desktop Firefox'], browserVersion: '83' } },
    { name: 'safari-min', use: { ...devices['Desktop Safari'], browserVersion: '14' } }
  ]
});
```

For the lowest-friction check, use `browserslist` in your build:

```
/* .browserslistrc */
>= 0.5%
last 2 versions
not dead
not IE 11
```

Then both Babel and PostCSS will only transpile what's actually needed.
