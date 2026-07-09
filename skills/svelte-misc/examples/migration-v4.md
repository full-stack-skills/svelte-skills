# Svelte 3 → 4 Migration Examples

Concrete before/after snippets for the Svelte 3 → 4 upgrade. Run the migration script for the mechanical changes, then check the items below that need a human eye.

```bash
npx svelte-migrate@latest svelte-4
```

---

## 1. Custom Elements: `tag` → `customElement`

The `tag` option in `<svelte:options>` is replaced by a richer `customElement` object.

```svelte
<!-- BEFORE (Svelte 3) -->
<svelte:options tag="my-component" />

<script>
  export let name;
</script>

<h1>Hello {name}!</h1>
```

```svelte
<!-- AFTER (Svelte 4+) -->
<svelte:options customElement="my-component" />

<script>
  let { name = 'world' } = $props();
</script>

<h1>Hello {name}!</h1>
```

The migration script handles this automatically. Property update timing changed slightly — the inner Svelte component is created in the next tick after `connectedCallback`.

---

## 2. `SvelteComponentTyped` → `SvelteComponent`

`SvelteComponentTyped` is deprecated; `SvelteComponent` has all its typing now.

```ts
// BEFORE
import { SvelteComponentTyped } from 'svelte';
export class Foo extends SvelteComponentTyped<{ aProp: string }> {}

// AFTER
import { SvelteComponent } from 'svelte';
export class Foo extends SvelteComponent<{ aProp: string }> {}
```

If you used `SvelteComponent` as a generic instance type before, you may need to add `<any>`:

```svelte
<!-- BEFORE -->
<script>
  import ComponentA from './ComponentA.svelte';
  import ComponentB from './ComponentB.svelte';
  import { SvelteComponent } from 'svelte';
  let component: typeof SvelteComponent;
</script>
```

```svelte
<!-- AFTER -->
<script>
  import ComponentA from './ComponentA.svelte';
  import ComponentB from './ComponentB.svelte';
  import { SvelteComponent } from 'svelte';
  let component: typeof SvelteComponent<any>;
</script>
```

---

## 3. Stricter `createEventDispatcher` Payload Types

The dispatcher now checks optional/required/no-argument payloads under strict mode.

```ts
// BEFORE (Svelte 3) — dispatcher accepts anything
import { createEventDispatcher } from 'svelte';
const dispatch = createEventDispatcher<{
  optional: number | null;
  required: string;
  noArgument: null;
}>();

dispatch('optional');
dispatch('required');          // could omit detail
dispatch('noArgument', 'surprise'); // could add extra arg
```

```ts
// AFTER (Svelte 4 strict) — flagged under TypeScript strict mode
dispatch('optional');
dispatch('required');          // ERROR: missing required argument
dispatch('noArgument', 'surprise'); // ERROR: cannot pass extra argument
```

> In Svelte 5 the recommended replacement is callback props — `createEventDispatcher` is legacy. Use it during the 3→4 migration, then convert to callback props in a follow-up 4→5 migration.

---

## 4. Stricter `Action` and `ActionReturn`

`Action` now has a default parameter of `undefined`. Type the generic if your action needs a parameter.

```ts
// BEFORE — params implicitly `any`
const action: Action = (node, params) => {
  console.log(params.foo); // no error
};
```

```ts
// AFTER — params must be typed
const action: Action<HTMLElement, { foo: string }> = (node, params) => {
  console.log(params.foo); // OK
};
```

The Svelte 4 migration script rewrites these automatically.

---

## 5. `onMount` Won't Accept an Async Returned Cleanup

A function passed to `onMount` is treated as cleanup only when returned synchronously. Async returners used to silently leak.

```js
// BEFORE — looked fine, but cleanup was never called
onMount(async () => {
  const something = await foo();
  // ...
  return () => someCleanup(); // never runs
});
```

```js
// AFTER — synchronous outer function, async work inside
onMount(() => {
  foo().then(something => {
    // ...
  });
  return () => someCleanup(); // runs on destroy
});
```

---

## 6. Transitions Are Local by Default

A transition no longer plays if a parent control flow block (not the direct parent) is created/destroyed. Use the `|global` modifier to keep the old behavior.

```svelte
<!-- Svelte 3 — 'slide' played on either `show` or `success` change -->
{#if show}
  ...
  {#if success}
    <p in:slide>Success</p>
  {/if}
{/if}
```

```svelte
<!-- Svelte 4 — 'slide' only plays on `success` change (local by default) -->
{#if show}
  ...
  {#if success}
    <p in:slide>Success</p>
  {/if}
{/if}
```

```svelte
<!-- Svelte 4 — to keep old behavior, add |global -->
{#if show}
  ...
  {#if success}
    <p in:slide|global>Success</p>
  {/if}
{/if}
```

The migration script will add `|global` to your existing transitions if it judges that the surrounding context is the issue — verify its output.

---

## 7. Default Slot Bindings Don't Cross Into Named Slots

```svelte
<!-- Nested.svelte -->
<script>
  export let count = 0;
</script>

<slot {count} />
<slot name="bar" />
```

```svelte
<!-- BEFORE (Svelte 3) — `count` leaked into named slot too -->
<Nested let:count>
  <p>default: {count}</p>
  <p slot="bar">bar: {count}</p>     <!-- count was available -->
</Nested>
```

```svelte
<!-- AFTER (Svelte 4) — `count` only available in default slot -->
<Nested let:count>
  <p>default: {count}</p>
  <p slot="bar">bar: {count}</p>     <!-- count is NOT available here -->
</Nested>
```

If you need `count` in both, pass it explicitly or use a prop:

```svelte
<Nested let:count count={count}>
  <p>default: {count}</p>
  <p slot="bar">bar: {count}</p>
</Nested>
```

---

## 8. Preprocessor Order Changed

Preprocessors are now applied in order, and within one preprocessor, markup → script → style. This can affect things if you have a chain of preprocessors.

```js
// Old (Svelte 3) order: all markup, then all script, then all style
// New (Svelte 4) order: each preprocessor runs markup, script, style before the next

import { preprocess } from 'svelte/compiler';

await preprocess(source, [preprocessorA, preprocessorB]);
// Svelte 3: A.markup, B.markup, A.script, B.script, A.style, B.style
// Svelte 4: A.markup, A.script, A.style, B.markup, B.script, B.style
```

Reorder your preprocessors if you depended on the old sequence.

---

## 9. CJS Output Removed (Plus `svelte/register`)

Svelte 4 only emits ESM. The `svelte/register` hook (used to compile Svelte from CommonJS at runtime) is gone.

```js
// BEFORE — Svelte 3 / svelte/register from CJS
require('svelte/register');
const App = require('./App.svelte').default;
```

```js
// AFTER — use a bundler (esbuild, Vite, tsx, etc.)
import App from './App.svelte';
```

If you need to stay on CJS output, transpile Svelte's ESM to CJS in a post-build step (e.g. with esbuild's `format: 'cjs'`).

---

## 10. Minimum Versions Checklist

```json
{
  "engines": {
    "node": ">=16"
  },
  "devDependencies": {
    "svelte": "^4.0.0",
    "@sveltejs/kit": "^1.20.4",
    "@sveltejs/vite-plugin-svelte": "^2.4.1",
    "vite-plugin-svelte": "^2.4.1",
    "svelte-loader": "^3.1.8",
    "rollup-plugin-svelte": "^7.1.5",
    "typescript": "^5.0.0"
  }
}
```

For Rollup, ensure `@rollup/plugin-node-resolve` sets `browser: true`. For Webpack, add `"browser"` to `conditionNames`. If you skip this, `onMount` (and other lifecycle hooks) won't fire in the browser — SvelteKit and Vite handle it automatically.
