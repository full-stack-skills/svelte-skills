# Routing — Pages and Layouts Examples

File-system routing with `+page.svelte`, `+page.js`, `+page.server.js`, `+layout.svelte`, `+layout.js`, `+layout.server.js`. All snippets are based on the official SvelteKit docs.

## 1. Simplest page

```svelte
<!-- src/routes/+page.svelte -->
<h1>Hello!</h1>
<a href="/about">About</a>
```

## 2. Page receiving data from `+page.js`

```js
// src/routes/blog/[slug]/+page.js
import { error } from '@sveltejs/kit';

/** @type {import('./$types').PageLoad} */
export function load({ params }) {
  if (params.slug === 'hello-world') {
    return {
      title: 'Hello world!',
      content: 'Welcome to our blog.'
    };
  }
  error(404, 'Not found');
}
```

```svelte
<!-- src/routes/blog/[slug]/+page.svelte -->
<script>
  /** @type {import('./$types').PageProps} */
  let { data } = $props();
</script>

<h1>{data.title}</h1>
<div>{@html data.content}</div>
```

## 3. Server-only load (database)

```js
// src/routes/blog/[slug]/+page.server.js
import { error } from '@sveltejs/kit';

/** @type {import('./$types').PageServerLoad} */
export async function load({ params }) {
  const post = await getPostFromDatabase(params.slug);
  if (!post) error(404, 'Not found');
  return post;
}
```

## 4. Per-route prerender / ssr / csr flags

```js
// src/routes/about/+page.js
export const prerender = true;   // pre-render at build time
export const ssr = true;         // server-render at runtime
export const csr = true;         // hydrate in browser
```

Accepts `true` | `false` | `'auto'`. Defaults are `true` for all three.

## 5. Root layout

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  let { children } = $props();
</script>
<nav>
  <a href="/">Home</a>
  <a href="/about">About</a>
</nav>
{@render children()}
```

## 6. Layout with data

```svelte
<!-- src/routes/settings/+layout.svelte -->
<script>
  /** @type {import('./$types').LayoutProps} */
  let { data, children } = $props();
</script>

<h1>Settings</h1>
<div class="submenu">
  {#each data.sections as section}
    <a href="/settings/{section.slug}">{section.title}</a>
  {/each}
</div>
{@render children()}
```

```js
// src/routes/settings/+layout.js
/** @type {import('./$types').LayoutLoad} */
export function load() {
  return {
    sections: [
      { slug: 'profile', title: 'Profile' },
      { slug: 'notifications', title: 'Notifications' }
    ]
  };
}
```

## 7. Nested layout inherits parent

```tree
src/routes/
├ +layout.svelte         # root: nav
└ settings/
   ├ +layout.svelte      # sub: settings submenu
   ├ profile/+page.svelte
   └ notifications/+page.svelte
```

`profile` and `notifications` get BOTH the root nav AND the settings submenu. Layout data from `settings/+layout.js` is also available to its children.

## 8. Layout `data` cascades down

```js
// src/routes/+layout.js
/** @type {import('./$types').LayoutLoad} */
export function load() {
  return { user: { name: 'Ada' } };
}
```

```svelte
<!-- src/routes/profile/+page.svelte -->
<script>
  /** @type {import('./$types').PageProps} */
  let { data } = $props();
</script>
<p>Hello {data.user.name}</p>   <!-- 'Ada' -->
```

## 9. Layout with form actions

```js
// src/routes/login/+page.server.js
import { fail, redirect } from '@sveltejs/kit';

/** @type {import('./$types').Actions} */
export const actions = {
  default: async ({ request }) => {
    const data = await request.formData();
    const email = data.get('email');
    const password = data.get('password');
    if (password !== 'hunter2') return fail(401, { email, error: 'Bad password' });
    throw redirect(303, '/');
  }
};
```

```svelte
<!-- src/routes/login/+page.svelte -->
<script>
  /** @type {import('./$types').PageProps} */
  let { form } = $props();
</script>
<form method="POST">
  <input name="email" value={form?.email ?? ''} />
  <input name="password" type="password" />
  {#if form?.error}<p>{form.error}</p>{/if}
  <button>Log in</button>
</form>
```

## 10. Dynamic params

```tree
src/routes/
└ blog/
   └ [slug]/+page.svelte
```

```svelte
<script>
  /** @type {import('./$types').PageProps} */
  let { params } = $props();
</script>
<h1>Post {params.slug}</h1>
```

## 11. Rest param (greedy)

```tree
src/routes/
└ files/
   └ [...path]/+page.js
```

```js
/** @type {import('./$types').PageLoad} */
export function load({ params }) {
  // /files/a/b/c -> params.path === 'a/b/c'
  return { path: params.path };
}
```

## 12. Optional param

```tree
src/routes/
└ blog/
   └ [[slug]]/+page.svelte
```

Matches `/blog` AND `/blog/anything`.

## 13. Force server navigation per-link

```svelte
<a href="/heavy-page" data-sveltekit-reload>Heavy page</a>
```

## 14. Disabling JS for a route

```js
// src/routes/legacy/+page.js
export const csr = false;
```

All links become full reloads, no hydration overhead.

## 15. Programmatic navigation

```svelte
<script>
  import { goto } from '$app/navigation';

  function login() {
    return goto('/login');
  }
</script>
<button onclick={login}>Log in</button>
```

```svelte
<script>
  import { beforeNavigate, afterNavigate } from '$app/navigation';

  beforeNavigate(({ from, to }) => console.log('nav', from?.url.pathname, '->', to?.url.pathname));
  afterNavigate(() => console.log('done'));
</script>
```