# Remote Functions Examples

Type-safe client/server communication via `*.remote.js` files. Opt in with `kit.experimental.remoteFunctions` and `compilerOptions.experimental.async`.

## 1. Enable remote functions

```js
// svelte.config.js
export default {
  kit: { experimental: { remoteFunctions: true } },
  compilerOptions: { experimental: { async: true } }
};
```

## 2. Basic query

```js
// src/routes/blog/data.remote.js
import { query } from '$app/server';
import * as db from '$lib/server/database';

export const getPosts = query(async () => {
  return await db.sql`SELECT title, slug FROM post ORDER BY published_at DESC`;
});
```

```svelte
<!-- src/routes/blog/+page.svelte -->
<script>
  import { getPosts } from './data.remote';
</script>

<h1>Recent posts</h1>
<ul>
  {#each await getPosts() as { title, slug }}
    <li><a href="/blog/{slug}">{title}</a></li>
  {/each}
</ul>
```

## 3. Query with validated argument

```js
import * as v from 'valibot';
import { error } from '@sveltejs/kit';
import { query } from '$app/server';
import * as db from '$lib/server/database';

export const getPost = query(v.string(), async (slug) => {
  const [post] = await db.sql`SELECT * FROM post WHERE slug = ${slug}`;
  if (!post) error(404, 'Not found');
  return post;
});
```

```svelte
<script>
  import { getPost } from '../data.remote';
  let { params } = $props();
  const post = $derived(await getPost(params.slug));
</script>

<h1>{post.title}</h1>
```

## 4. query.batch — solve n+1

```js
import * as v from 'valibot';
import { query } from '$app/server';
import * as db from '$lib/server/database';

export const getWeather = query.batch(v.string(), async (cityIds) => {
  const weather = await db.sql`SELECT * FROM weather WHERE city_id = ANY(${cityIds})`;
  const lookup = new Map(weather.map((w) => [w.city_id, w]));
  return (cityId) => lookup.get(cityId);
});
```

Multiple `await getWeather(cityId)` calls in the same macrotask collapse into one DB query.

## 5. query.live — real-time stream

```js
import { query } from '$app/server';

export const getTime = query.live(async function* () {
  while (true) {
    yield new Date();
    await new Promise((f) => setTimeout(f, 1000));
  }
});
```

```svelte
<script>
  import { getTime } from './time.remote.js';
  const time = getTime();
</script>

<p>{await time}</p>
<p>connected: {time.connected}</p>
<button onclick={() => time.reconnect()}>Reconnect</button>
```

Note: do NOT cache `no-store` responses in a service worker — the stream stays open.

## 6. form — progressive-enhanced mutation

```js
import * as v from 'valibot';
import { error, redirect } from '@sveltejs/kit';
import { query, form } from '$app/server';
import * as db from '$lib/server/database';
import * as auth from '$lib/server/auth';

export const createPost = form(
  v.object({
    title: v.pipe(v.string(), v.nonEmpty()),
    content: v.pipe(v.string(), v.nonEmpty())
  }),
  async ({ title, content }) => {
    const user = await auth.getUser();
    if (!user) error(401, 'Unauthorized');
    const slug = title.toLowerCase().replace(/ /g, '-');
    await db.sql`INSERT INTO post (slug, title, content) VALUES (${slug}, ${title}, ${content})`;
    redirect(303, `/blog/${slug}`);
  }
);
```

```svelte
<!-- src/routes/blog/new/+page.svelte -->
<script>
  import { createPost } from '../data.remote';
</script>

<form {...createPost}>
  <label>
    Title
    <input {...createPost.fields.title.as('text')} />
  </label>
  <label>
    Content
    <textarea {...createPost.fields.content.as('text')}></textarea>
  </label>
  <button>Publish!</button>
</form>
```

## 7. Form validation display

```svelte
<form {...createPost}>
  {#each createPost.fields.title.issues() as issue}
    <p class="issue">{issue.message}</p>
  {/each}
  <input {...createPost.fields.title.as('text')} />
</form>
```

```svelte
<form {...createPost} oninput={() => createPost.validate()}>
  <!-- validates on every keystroke -->
</form>
```

## 8. Form with preflight

```svelte
<script>
  import * as v from 'valibot';
  import { createPost } from '../data.remote';
  const schema = v.object({
    title: v.pipe(v.string(), v.nonEmpty()),
    content: v.pipe(v.string(), v.nonEmpty())
  });
</script>

<form {...createPost.preflight(schema)}>
  <!-- client-side blocks submit before hitting server -->
</form>
```

## 9. Form with custom enhance

```svelte
<script>
  import { createPost } from '../data.remote';
  import { showToast } from '$lib/toast';
</script>

<form {...createPost.enhance(async (form) => {
  try {
    if (await form.submit()) {
      form.element.reset();
      showToast('Published!');
    } else {
      showToast('Invalid!');
    }
  } catch (e) {
    showToast('Error');
  }
})}>
  <!-- ... -->
</form>
```

## 10. Form with sensitive fields (underscore prefix)

```svelte
<form {...register}>
  <input {...register.fields.username.as('text')} />
  <input {...register.fields._password.as('password')} />
  <button>Sign up</button>
</form>
```

Fields starting with `_` are NOT sent back to the user on validation failure.

## 11. Form for multiple instances (.for)

```svelte
{#each await getTodos() as todo}
  {@const modify = modifyTodo.for(todo.id)}
  <form {...modify}>
    <input {...modify.fields.description.as('text', todo.description)} />
    <button disabled={!!modify.pending}>save</button>
  </form>
{/each}
```

## 12. command — imperative mutation

```js
import * as v from 'valibot';
import { query, command } from '$app/server';
import * as db from '$lib/server/database';

export const getLikes = query(v.string(), async (id) => {
  const [row] = await db.sql`SELECT likes FROM item WHERE id = ${id}`;
  return row.likes;
});

export const addLike = command(v.string(), async (id) => {
  await db.sql`UPDATE item SET likes = likes + 1 WHERE id = ${id}`;
});
```

```svelte
<script>
  import { getLikes, addLike } from './likes.remote';
  let { item } = $props();
</script>

<button onclick={async () => { await addLike(item.id); }}>
  like
</button>
<p>likes: {await getLikes(item.id)}</p>
```

## 13. Single-flight mutation (server-driven refresh)

```js
export const createPost = form(v.object({/* ... */}), async (data) => {
  // ... insert ...
  void getPosts().refresh();    // refresh all `getPosts()` on the client
  redirect(303, `/blog/${slug}`);
});

export const updatePost = form(
  v.object({ id: v.string() }),
  async (post) => {
    const result = await externalApi.update(post);
    getPost(post.id).set(result);  // push value directly
  }
);
```

## 14. Single-flight mutation (client-requested)

```js
import { requested } from '$app/server';

export const createPost = form(v.object({/* ... */}), async (data) => {
  // ... insert ...
  for (const { query } of requested(getPosts, 1)) {
    void query.refresh();
  }
  // or shorthand: await requested(getPosts, 1).refreshAll();
});
```

Client side calls `submit().updates(getPosts, getPosts({ filter: 'x' }), ...)`.

## 15. Reconnect live query in mutation

```js
export const markAllRead = form(
  v.object({ userId: v.string() }),
  async ({ userId }) => {
    // ... mutation ...
    getNotifications(userId).reconnect();
  }
);
```

## 16. prerender — build-time data

```js
import { prerender } from '$app/server';
import * as db from '$lib/server/database';

export const getPost = prerender(
  v.string(),
  async (slug) => { /* ... */ },
  { inputs: () => ['first-post', 'second-post', 'third-post'] }
);
```

`prerender` excludes from server bundle unless `dynamic: true`.

## 17. getRequestEvent inside remote

```js
import { getRequestEvent, query } from '$app/server';

export const getProfile = query(async () => {
  const { cookies } = getRequestEvent();
  return await db.findUser(cookies.get('session_id'));
});
```

## 18. Custom validation error

```js
// src/hooks.server.js
export function handleValidationError({ issues }) {
  return { message: 'Nice try, hacker!' };
}
```

Or opt out of validation entirely with `query('unchecked', async (arg) => ...)`.

## 19. Handle validation error programmatically

```js
import { invalid } from '@sveltejs/kit';

export const buyHotcakes = form(
  v.object({ qty: v.pipe(v.number(), v.minValue(1)) }),
  async (data, issue) => {
    try {
      await db.buy(data.qty);
    } catch (e) {
      if (e.code === 'OUT_OF_STOCK') {
        invalid(issue.qty("we don't have enough"));
      }
    }
  }
);
```

## 20. Redirect inside query/form/prerender (NOT command)

```js
export const getPost = query(v.string(), async (slug) => {
  if (someCondition) redirect(302, '/login');
  return post;
});
```

For `command`, return `{ redirect: location }` and handle in the client.