# Browser Support Reference

Svelte 5 targets the **Baseline 2020** browser set. The minimum versions below are derived from the browser APIs the Svelte runtime uses internally.

---

## Minimum Browser Versions

| Browser | Minimum version |
| - | - |
| Chrome / Edge | 87 |
| Firefox | 83 |
| Safari | 14 |
| Opera | 73 |
| Opera (Android) | 62 |
| Samsung Internet | 14.0 |
| Android WebView | 87 |
| Internet Explorer | **not supported** |

This equates to [Baseline](https://web-platform-dx.github.io/baseline/) 2020.

> This table covers Svelte itself. It does **not** include SvelteKit, third-party Svelte libraries, or your own application code. Verify the union of your app's requirements separately.

---

## Feature-Specific Exceptions

Most of the Svelte runtime fits in the table above. A few features need newer browser versions. The compiler warns when you use one of them.

| Feature | Chrome / Edge | Firefox | Safari |
| - | - | - | - |
| [`$state.snapshot`](/docs/svelte/$state#$state.snapshot) | 98 | 94 | 15.4 |
| [`bind:devicePixelContentBoxSize`](/docs/svelte/bind#Dimensions) | — | 93 | not supported |
| [`flip` from `svelte/animate`](/docs/svelte/svelte-animate#flip) | — | 126 | — |

If your browser target falls below the version for a feature you use, either:

- Polyfill / feature-detect the feature at runtime
- Avoid the feature
- Use a more compatible alternative (e.g. `$state.snapshot` → `JSON.parse(JSON.stringify(state))`)

---

## What "Baseline 2020" Means in Practice

Baseline 2020 includes:

- ES2018 (`async/await`, spread, rest, async iteration)
- ES2020 (optional chaining, nullish coalescing, `BigInt`)
- Native ESM (`<script type="module">`)
- Class fields
- Top-level `await` (in modules)
- `Proxy` / `Reflect`
- Native custom elements v1 (Web Components)
- Native shadow DOM
- `AbortController` / `fetch`
- `Intl` features

If you need older browsers (IE 11, legacy Edge), plan for polyfills and additional tooling (Babel, core-js).

---

## Polyfills for Older Browsers

The most commonly needed:

```bash
npm install -D core-js @webcomponents/webcomponentsjs
```

Conditionally load Web Components polyfill (only for browsers without it):

```html
<script>
  if (!('customElements' in window)) {
    import('@webcomponents/webcomponentsjs/custom-elements.min.js');
  }
</script>
```

---

## SSR

Server-side rendering is fully supported. The only caveat: avoid touching DOM globals (`window`, `document`, `localStorage`) during module load or component initialization. Use:

- `import { browser } from '$app/environment'` (SvelteKit)
- `<svelte:window>` and `<svelte:document>` (auto-gated for SSR)
- Effects (never run on the server)

```svelte
<script>
  import { onMount } from 'svelte';
  // BAD: runs on server, throws
  // window.addEventListener('resize', handler);

  // GOOD: only runs on client
  onMount(() => {
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  });
</script>
```

---

## Bundler Configuration

Svelte's runtime ships as ES2015+ JavaScript. Configure your bundler target accordingly.

### Vite / SvelteKit

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import { svelte } from '@sveltejs/vite-plugin-svelte';

export default defineConfig({
  plugins: [svelte()],
  build: {
    target: 'es2020' // or lower (es2015 = Baseline 2020)
  }
});
```

### TypeScript

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true
  }
}
```

`target: "ES2015"` (or higher) is required to keep ES classes as classes in compiled output, which `$state` class fields rely on.

---

## Detecting Feature Support at Runtime

For features with version-specific requirements, gate them with feature detection:

```svelte
<script>
  import { flip } from 'svelte/animate';

  const supportsFlip = typeof window !== 'undefined' &&
    CSS.supports('animation-timeline', 'view()');
</script>

{#if supportsFlip}
  <ul>
    {#each items as item (item.id)}
      <li animate:flip={{ duration: 300 }}>{item.text}</li>
    {/each}
  </ul>
{/if}
```

---

## Testing Older Browsers

Use Playwright with browser-version pinning:

```ts
// playwright.config.ts
export default defineConfig({
  projects: [
    { name: 'chrome-min', use: { ...devices['Desktop Chrome'], browserVersion: '87' } },
    { name: 'firefox-min', use: { ...devices['Desktop Firefox'], browserVersion: '83' } },
    { name: 'safari-min', use: { ...devices['Desktop Safari'], browserVersion: '14' } }
  ]
});
```

Use `browserslist` to limit Babel / PostCSS transpilation to what you actually support:

```
/* .browserslistrc */
>= 0.5%
last 2 versions
not dead
not IE 11
```

---

## Quick Reference

| Need | Tool / Pattern |
|------|----------------|
| Target specific browsers | `browserslist` in root or `package.json` |
| Polyfill Web Components | `@webcomponents/webcomponentsjs` (conditional load) |
| Avoid SSR errors | `<svelte:window>`, `onMount`, `$effect`, or `browser` check |
| Use `$state.snapshot` on older browsers | Fallback to `structuredClone` / `JSON` polyfill |
| Test the minimum versions | Playwright with `browserVersion` pinning |
