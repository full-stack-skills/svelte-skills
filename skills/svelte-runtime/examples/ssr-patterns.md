# Server-Side Rendering (SSR) Patterns

Practical examples for rendering Svelte components on the server with `render()`, async patterns, and hydration.

## 1. Basic render() Usage

Generate static HTML from a Svelte component:

```js
// server.js
import { render } from 'svelte/server';
import App from './App.svelte';

export async function renderPage(url) {
  const { head, body } = await render(App, {
    props: {
      url,
      user: null
    }
  });

  return `<!DOCTYPE html>
<html lang="en">
<head>${head}</head>
<body>
  <div id="app">${body}</div>
</body>
</html>`;
}
```

```svelte
<!-- App.svelte -->
<script>
  let { url, user } = $props();
</script>

<nav>
  <a href="/" class:active={url === '/'}>Home</a>
  <a href="/about" class:active={url === '/about'}>About</a>
  {#if user}
    <span>Welcome, {user.name}</span>
  {/if}
</nav>

<main>
  <h1>Hello World</h1>
</main>
```

**Return value:**
```js
{
  head: '<link rel="stylesheet" href="/app.css">...',
  body: '<nav>...</nav><main><h1>Hello World</h1></main>'
}
```

## 2. Async render()

Handle async data within components:

```js
// +page.server.js (SvelteKit)
import { render } from 'svelte/server';

export async function load({ fetch, params }) {
  // Fetch data
  const userData = await fetch(`/api/users/${params.id}`).then(r => r.json());
  const posts = await fetch(`/api/users/${params.id}/posts`).then(r => r.json());

  // Render with async data
  const { head, body } = await render(UserProfile, {
    props: {
      user: userData,
      posts
    }
  });

  return {
    head,
    body,
    user: userData, // Serializable for hydration
    posts
  };
}
```

```svelte
<!-- UserProfile.svelte -->
<script>
  let { user, posts = [] } = $props();
</script>

<h1>{user.name}</h1>
<p>{user.bio}</p>

<h2>Posts ({posts.length})</h2>
<ul>
  {#each posts as post}
    <li>
      <a href="/post/{post.slug}">{post.title}</a>
    </li>
  {/each}
</ul>
```

## 3. Streaming SSR

Stream content progressively to the client:

**SvelteKit implementation:**

```js
// +page.server.js
export async function load({ fetch }) {
  // Start fetching immediately
  const criticalData = fetch('/api/critical').then(r => r.json());

  return {
    // Use deferred for non-critical data
    deferred: {
      analytics: fetch('/api/analytics').then(r => r.json()),
      recommendations: fetch('/api/recommendations').then(r => r.json())
    },
    // Await critical data
    critical: await criticalData
  };
}
```

```svelte
<!-- +page.svelte -->
<script>
  let { data } = $props();
</script>

<!-- Renders immediately with critical data -->
<main>
  <h1>{data.critical.title}</h1>
  {@render data.deferred.analytics.stream()}
</main>

<!-- Shows loading state, fills when resolved -->
{#await data.deferred.recommendations}
  <div class="loading">Loading recommendations...</div>
{:then recommendations}
  <aside>
    {#each recommendations as rec}
      <p>{rec.title}</p>
    {/each}
  </aside>
{/await}
```

**Custom streaming with render():**

```js
import { render } from 'svelte/server';

export async function streamPage() {
  const stream = new ReadableStream({
    async start(controller) {
      // Flush initial HTML
      const { head, body } = await render(Shell, { props: { status: 'loading' } });
      controller.enqueue(`<html><head>${head}</head><body>${body}`);

      // Stream content as it loads
      const data = await fetch('/api/content').then(r => r.json());
      const { body: contentBody } = await render(Content, { props: data });
      controller.enqueue(contentBody);

      controller.close();
    }
  });

  return new Response(stream, {
    headers: { 'Content-Type': 'text/html' }
  });
}
```

## 4. Hydration Mismatch Prevention

Ensure SSR and CSR produce identical output:

**Problem:** Mismatches cause hydration errors

```js
// WRONG: Different output between server and client
let timestamp = Date.now(); // Different each run!

// RIGHT: Use consistent values
let userId = $props().userId; // Passed consistently
```

**Fix with hydration-safe patterns:**

```svelte
<!-- UserBadge.svelte - hydration safe -->
<script>
  let { user } = $props();

  // Use stable IDs, not random values
  let badgeId = $derived(`badge-${user.id}`);

  // Derive display values from props, don't generate randomly
  let initials = $derived(
    user.name.split(' ').map(n => n[0]).join('').toUpperCase()
  );
</script>

<div id={badgeId} class="badge">
  <span class="initials">{initials}</span>
</div>
```

**Server vs Client consistency:**

```js
// server.js - Use same data on both sides
import { render } from 'svelte/server';

export async function renderUserPage(userId) {
  const user = await db.users.findById(userId);

  // Render on server
  const { head, body } = await render(UserPage, { props: { user } });

  return {
    html: `<!DOCTYPE html>...${body}...</html>`,
    // Serialize same data for client hydration
    dehydratedState: {
      user: { id: user.id, name: user.name, role: user.role }
    }
  };
}
```

```js
// client.js - Use dehydrated state
import { hydrate } from 'svelte';
import UserPage from './UserPage.svelte';

const state = window.__DEHYDRATED_STATE__;

const app = hydrate(UserPage, {
  target: document.querySelector('#app'),
  props: { user: state.user }
});
```

## 5. flushSync in Tests

Force synchronous updates when testing SSR output:

```js
import { render, flushSync } from 'svelte';
import { expect, test } from 'vitest';
import UserList from './UserList.svelte';

test('renders user list', async () => {
  const users = [
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' }
  ];

  const { head, body } = await render(UserList, { props: { users } });

  // Verify server output
  expect(body).toContain('Alice');
  expect(body).toContain('Bob');
});

test('hydration preserves state', async () => {
  const users = [{ id: 1, name: 'Charlie' }];

  // Simulate client hydration
  const target = document.createElement('div');
  target.innerHTML = await render(UserList, { props: { users } }).then(r => r.body);

  const { hydrate } = await import('svelte');
  const app = hydrate(UserList, {
    target,
    props: { users }
  });

  // Modify state and flush
  app.increment?.();
  flushSync();

  expect(target.innerHTML).toContain('Charlie');
});
```

## 6. CSP Nonce Handling

Add nonce to scripts for Content Security Policy:

```js
// server.js
import { render } from 'svelte/server';
import crypto from 'crypto';

export async function renderWithCSP(App, props) {
  // Generate nonce
  const nonce = crypto.randomBytes(16).toString('base64');

  const { head, body } = await render(App, {
    props,
    csp: {
      nonce,
      // Or use hash mode (auto-generates nonce from script content)
      // hash: true
    }
  });

  // Inject nonce into scripts
  const headWithNonce = head.replace(
    /<script([^>]*)>/g,
    `<script${nonce ? ` nonce="${nonce}"` : ''}$1>`
  );

  return {
    head: headWithNonce,
    body,
    nonce
  };
}
```

```svelte
<!-- App.svelte with inline event handlers -->
<script>
  let count = $state(0);
</script>

<button onclick={() => count++}>
  Count: {count}
</button>

<!-- Scripts in head use nonce automatically -->
```

**CSP Header:**
```
Content-Security-Policy: script-src 'self' 'nonce-abc123';
```

## 7. Component with Hydratable Data

Use `hydratable()` for data that exists on server and client:

```js
// +page.server.js
import { render } from 'svelte/server';

export async function load({ fetch }) {
  const user = await fetch('/api/me').then(r => r.json());

  return { user };
}
```

```svelte
<!-- +page.svelte -->
<script>
  import { hydratable } from 'svelte';

  let { data } = $props();

  // user exists server-side, available client-side without refetch
  const user = hydratable('user', () => data.user);
</script>

<h1>Welcome, {user.name}</h1>
```

## Summary

| Pattern | When to Use |
|---------|-------------|
| Basic `render()` | Static pages, SEO content |
| Async `render()` | Data-dependent pages |
| Streaming | Large pages, progressive loading |
| Hydration | Interactive SSR content |
| CSP nonce | Security-sensitive deployments |
| `hydratable()` | Shared server/client data |
