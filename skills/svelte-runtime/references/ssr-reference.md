# Server-Side Rendering (SSR) Reference

Complete technical reference for Svelte 5's SSR APIs including `render()`, hydration, async patterns, and CSP.

## render() API

### Signature

```ts
function render<T extends Component>(
  component: T,
  options?: {
    props?: Record<string, unknown>;
    csp?: {
      nonce?: string;
      hash?: boolean;
    };
  }
): Promise<{ head: string; body: string }>;
```

### Return Value

```ts
{
  head: string;  // HTML for <head> (styles, scripts, meta)
  body: string;  // HTML for component render
}
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `component` | `Component` | Yes | Svelte component to render |
| `props` | `Record<string, unknown>` | No | Props passed to component |
| `csp.nonce` | `string` | No | Nonce string for script tags |
| `csp.hash` | `boolean` | No | Auto-generate nonce from content |

### Basic Usage

```js
import { render } from 'svelte/server';
import App from './App.svelte';

const { head, body } = await render(App, {
  props: {
    user: { name: 'Grace Hopper', role: 'Computer Scientist' }
  }
});

// Use in HTML template
const html = `
<!DOCTYPE html>
<html>
<head>${head}</head>
<body>
  <div id="app">${body}</div>
</body>
</html>
`;
```

### What Goes in head vs body

| `head` | `body` |
|--------|--------|
| `<style>` tags | Component template HTML |
| `<link>` tags | `<script>` tags (inline) |
| `<script>` tags (external) | |
| `<meta>` tags | |
| `<title>` | |

## Async SSR

### Async Components

Components can use async data in their renders:

```svelte
<!-- Dashboard.svelte -->
<script>
  let { data } = $props();
  // data.users, data.stats loaded from server
</script>

<h1>Dashboard</h1>
<div class="stats">
  {#each data.stats as stat}
    <div class="stat">{stat.label}: {stat.value}</div>
  {/each}
</div>
```

### Async render()

The `render()` function itself is always async to support streaming:

```js
const { head, body } = await render(HeavyComponent, {
  props: { data: await fetchData() }
});
```

### Data Fetching Patterns

**Server-side data loading:**

```js
// +page.server.js (SvelteKit)
import { render } from 'svelte/server';

export async function load({ fetch, params }) {
  // Parallel data fetching
  const [user, posts] = await Promise.all([
    fetch(`/api/users/${params.id}`).then(r => r.json()),
    fetch(`/api/users/${params.id}/posts`).then(r => r.json())
  ]);

  return {
    head: '', // Populated by render
    body: '', // Populated by render
    user,
    posts
  };
}
```

**Passing to render:**

```js
// In a custom server setup
import { render } from 'svelte/server';
import Dashboard from './Dashboard.svelte';

export async function renderDashboard(data) {
  return render(Dashboard, {
    props: { data }
  });
}
```

## Streaming SSR

### How Streaming Works

1. Initial shell HTML is sent
2. Async holes (`{@render deferred.stream()}`) wait
3. When data resolves, content is streamed in
4. Client hydrates progressively

### Implementation

```js
import { render } from 'svelte/server';

export async function streamPage(data) {
  const stream = new ReadableStream({
    async start(controller) {
      const encoder = new TextEncoder();

      // Send initial HTML shell
      const shell = `<!DOCTYPE html>
<html>
<head><title>Loading...</title></head>
<body>
<div id="app"><main>Loading...</main>`;
      controller.enqueue(encoder.encode(shell));

      // Stream in deferred content
      const resolved = await data.deferred.content;
      const { body } = await render(Content, { props: { content: resolved } });
      controller.enqueue(encoder.encode(body));

      // Close
      const closing = `</div></body></html>`;
      controller.enqueue(encoder.encode(closing));
      controller.close();
    }
  });

  return new Response(stream, {
    headers: { 'Content-Type': 'text/html; charset=utf-8' }
  });
}
```

### SvelteKit Streaming

SvelteKit handles streaming automatically with `deferred` in load functions:

```js
// +page.server.js
export async function load({ fetch }) {
  return {
    // Resolved before navigation
    user: await fetch('/api/me').then(r => r.json()),

    // Deferred - streams after initial render
    deferred: {
      feed: fetch('/api/feed').then(r => r.json())
    }
  };
}
```

```svelte
<!-- +page.svelte -->
<script>
  let { data } = $props();
</script>

<h1>Welcome {data.user.name}</h1>

<!-- Streams when ready -->
{#await data.deferred.feed}
  <div class="loading">Loading feed...</div>
{:then feed}
  <feed>
    {#each feed.items as item}
      <item>{item.title}</item>
    {/each}
  </feed>
{/await}
```

## CSP Nonce Handling

### What is CSP?

Content Security Policy prevents XSS by controlling script sources:

```
Content-Security-Policy: script-src 'self' 'nonce-abc123';
```

### Automatic Nonce Injection

```js
import { render } from 'svelte/server';
import crypto from 'crypto';

export function renderPage(Component, props) {
  const nonce = crypto.randomBytes(16).toString('base64');

  const { head, body } = render(Component, {
    props,
    csp: { nonce }
  });

  // Inject nonce into all script tags in head
  const secureHead = head.replace(
    /<script(?![^>]*nonce)([^>]*)>/g,
    `<script nonce="${nonce}"$1>`
  );

  return { head: secureHead, body, nonce };
}
```

### Hash Mode

Generate nonce based on script content:

```js
const { head, body } = render(Component, {
  props,
  csp: { hash: true }
});
```

### Script Tag Types

| Type | Nonce Behavior |
|------|----------------|
| External `<script src="...">` | Requires nonce in CSP header |
| Inline `<script>...</script>` | Can use hash or nonce |
| Event handlers (onclick) | Don't need nonce (not script) |

## Hydration and Re-execution

### How Hydration Works

1. Server renders HTML with component structure
2. Client receives HTML (no interactivity yet)
3. `hydrate()` attaches event listeners
4. Component re-executes initialization
5. State is re-derived from props, not re-fetched

### Hydration Process

```js
// Server
const { head, body } = await render(UserPage, {
  props: { userId: 42 }
});
// body = '<div>User: Ada</div>'

// Client
const app = hydrate(UserPage, {
  target: document.querySelector('#app'),
  props: { userId: 42 } // Same props!
});
```

### Re-execution Safety

**Safe to re-execute:**
- `$state` initialization from props
- `$derived` computations
- `onMount` / `$effect` setup

**NOT re-executed:**
- Server-only code (inside `if (browser)` checks)
- Data fetching that should use SSR state

### State Consistency

**Must match between SSR and CSR:**

```js
// Server: userId=42, user.name="Ada"
// Client: MUST pass same userId=42
//        If different: hydration mismatch error!

hydrate(Component, {
  props: { userId: 42 } // Same as server
});
```

### Hydration Mismatch Errors

**Causes:**
- Different props passed to hydrate vs render
- DOM structure changed between server and client
- External modifications to DOM

**Error message:**
```
Hydration failed because the initial UI does not match what was rendered on the server.
```

**Fix checklist:**
1. Same props passed to `render()` and `hydrate()`
2. No DOM manipulation between SSR and hydration
3. Server and client render same structure
4. Use `hydratable()` for consistent data

## flushSync in SSR Context

### Usage

```js
import { render, flushSync } from 'svelte';

test('SSR output matches expectations', async () => {
  const { body } = await render(MyComponent, {
    props: { items: ['a', 'b', 'c'] }
  });

  flushSync(); // No-op in SSR context (no reactive updates)

  expect(body).toContain('a');
  expect(body).toContain('b');
});
```

### Key Difference from Client

| Context | flushSync Effect |
|---------|------------------|
| Client | Forces pending `$effect` updates to run synchronously |
| Server | No-op - no reactive updates during SSR |

### SSR is Synchronous

SSR renders are synchronous - there's no microtask queue:

```js
// Server-side
const { body } = await render(Component, { props });

// All $state is already resolved
// All $derived is already computed
// body contains final HTML
```

### Testing SSR Output

```js
import { render } from 'svelte';
import { expect, test } from 'vitest';

test('renders accessible markup', async () => {
  const { head, body } = await render(AccessibleForm, {
    props: { fields: ['name', 'email'] }
  });

  // Check for aria attributes
  expect(body).toContain('aria-label');
  expect(body).toContain('role="form"');

  // Check for proper label associations
  expect(body).toContain('for="name"');
  expect(body).toContain('id="name"');
});
```

## Summary Table

| API | Description |
|-----|-------------|
| `render()` | Generate HTML on server, returns `{ head, body }` |
| `hydrate()` | Activate SSR HTML on client |
| `flushSync()` | No-op in SSR; forces sync updates on client |
| `tick()` | No-op in SSR; waits for DOM updates on client |
| `hydratable()` | Mark data for SSR/CSR sharing |

## Security Checklist

- [ ] Use CSP nonce for all inline scripts
- [ ] Validate props don't contain user-generated HTML
- [ ] Sanitize data before passing to `render()`
- [ ] Use `hash: true` when nonce rotation is complex
- [ ] Test hydration mismatch errors in CI
