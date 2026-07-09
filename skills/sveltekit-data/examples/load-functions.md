# Examples: SvelteKit Load Functions

17 个常见 load 函数场景，全部基于 `/tmp/sveltekit-llms.txt` 文档化模式。

## 1. Universal page load（基础）

`src/routes/blog/[slug]/+page.js`

```js
/** @type {import('./$types').PageLoad} */
export function load({ params }) {
  return {
    post: {
      title: `Title for ${params.slug}`,
      content: `Content for ${params.slug}`
    }
  };
}
```

页面通过 `data.post` 接收，类型由 `./$types` 的 `PageData` 自动推导。

## 2. Server page load（DB / 私密凭据）

`src/routes/blog/[slug]/+page.server.js`

```js
import * as db from '$lib/server/database';

/** @type {import('./$types').PageServerLoad} */
export async function load({ params }) {
  return { post: await db.getPost(params.slug) };
}
```

注意类型从 `PageLoad` → `PageServerLoad`——server load 可用 `cookies` / `locals` / `request`。

## 3. Layout load（共享数据）

`src/routes/blog/[slug]/+layout.server.js`

```js
import * as db from '$lib/server/database';

/** @type {import('./$types').LayoutServerLoad} */
export async function load() {
  return { posts: await db.getPostSummaries() };
}
```

返回的 `posts` 在该 layout 下所有页面共享，子 page 也能访问 `data.posts`。

## 4. page.data（父 layout 访问子数据）

`src/routes/+layout.svelte`

```svelte
<script>
  import { page } from '$app/state';
  let { children } = $props();
</script>

<svelte:head>
  <title>{page.data.title}</title>
</svelte:head>
{@render children()}
```

> `$app/state` 自 SvelteKit 2.12 引入。SvelteKit < 2.12 或 Svelte 4 用 `$app/stores` 的 `$page.data`。

## 5. Universal + Server 组合

```js
// +page.server.js —— 仅服务端，返回不敏感的 sessionId
export async function load({ locals }) {
  return { sessionId: locals.sessionId };
}

// +page.js —— 浏览器也能跑，能拿到 server 上下文
/** @type {import('./$types').PageLoad} */
export async function load({ data, fetch }) {
  const res = await fetch('/api/me', { headers: { 'x-session': data.sessionId } });
  return { me: await res.json() };
}
```

server load 先跑，返回值进 universal load 的 `data` 参数。

## 6. URL data（params/route/url）

`src/routes/a/[b]/[...c]/+page.js`

```js
/** @type {import('./$types').PageLoad} */
export function load({ url, route, params }) {
  // route.id = '/a/[b]/[...c]'
  // url.searchParams / url.pathname / url.origin
  // params.b = 'x', params.c = 'y/z'
  return { q: url.searchParams.get('q') };
}
```

`url.hash` SSR 不可用（服务端无 fragment）。

## 7. fetch with credentials（同源 / 子域）

```js
// +page.js
export async function load({ fetch, params }) {
  const res = await fetch(`/api/items/${params.id}`);
  return { item: await res.json() };
}
```

- 继承 page request 的 `cookie` / `authorization` headers
- SSR 时相对 URL 可用（原生 fetch 服务端需要绝对 URL）
- 内部 `+server.js` 调用直接走 handler，无 HTTP 开销
- SSR response 内联到 HTML，hydration 复用，无额外网络请求

## 8. Cookies（仅 server load）

`src/routes/+layout.server.js`

```js
import * as db from '$lib/server/database';

export async function load({ cookies }) {
  const sessionid = cookies.get('sessionid');
  return { user: await db.getUser(sessionid) };
}
```

`fetch` 只在**同源或子域**目标带 cookies。其他域需用 `handleFetch` hook。

## 9. setHeaders（转发 cache 头）

```js
// +page.js
export async function load({ fetch, setHeaders }) {
  const response = await fetch('https://cms.example.com/products.json');
  setHeaders({
    'cache-control': response.headers.get('cache-control'),
    age: response.headers.get('age')
  });
  return response.json();
}
```

约束：同名 header 只能设一次；不能设 `set-cookie`（用 `cookies.set`）。

## 10. parent()（访问父 load 数据）

```js
// src/routes/+layout.js
export function load() {
  return { a: 1 };
}

// src/routes/abc/+layout.js
export async function load({ parent }) {
  const { a } = await parent();
  return { b: a + 1 };  // 2
}

// src/routes/abc/+page.js
export async function load({ parent }) {
  const { a, b } = await parent();
  return { c: a + b };  // 3
}
```

`+page.js` 拿到的是**全部祖先**的合并数据，不只是直接父 layout。

## 11. 避免 parent() 瀑布

```js
// +page.js —— 错误：先 parent 再 getData 串行
export async function load({ params, parent }) {
  const parentData = await parent();
  const data = await getData(params);  // 等 parent 完才跑
  return { ...data, meta: { ...parentData.meta, ...data.meta } };
}

// 正确：getData 不依赖 parent，并行启动
export async function load({ params, parent }) {
  const data = await getData(params);
  const parentData = await parent();
  return { ...data, meta: { ...parentData.meta, ...data.meta } };
}
```

## 12. error() / redirect()

```js
import { error, redirect } from '@sveltejs/kit';

export function load({ locals }) {
  if (!locals.user) error(401, 'not logged in');
  if (!locals.user.isAdmin) error(403, 'not an admin');
}

export function load({ locals, url }) {
  if (!locals.user) {
    redirect(303, `/login?redirectTo=${encodeURIComponent(url.pathname + url.search)}`);
  }
}
```

`error` / `redirect` 都**直接抛**——SvelteKit 2.x 不再需要 `throw`（1.x 需要）。`redirect` **不要**放 `try` 块里。

## 13. Streaming（流式返回 Promise）

`src/routes/blog/[slug]/+page.server.js`

```js
export async function load({ params }) {
  return {
    post: await loadPost(params.slug),       // 关键数据先 await
    comments: loadComments(params.slug)      // 不 await → 单独流
  };
}
```

```svelte
<!-- +page.svelte -->
{#await data.comments}
  <p>Loading comments...</p>
{:then comments}
  {#each comments as c}<p>{c.content}</p>{/each}
{:catch error}
  <p>error: {error.message}</p>
{/await}
```

约束：JS 必须启用；非 Lambda/Firebase 平台；开始流后**不能**再 `setHeaders` / `redirect`；手写 Promise 加 `.catch(() => {})` 防止 unhandled rejection。

## 14. depends() + invalidate()

```js
// +page.js
export async function load({ fetch, depends }) {
  depends('app:random');  // 自定义 id，前缀约定 [a-z]:
  const r = await fetch('https://api.example.com/random-number');
  return { number: await r.json() };
}
```

```svelte
<script>
  import { invalidate, invalidateAll } from '$app/navigation';
  function rerun() {
    invalidate('app:random');
    invalidate('https://api.example.com/random-number');
    invalidate(url => url.href.includes('random-number'));
    invalidateAll();
  }
</script>
<button onclick={rerun}>refresh</button>
```

**server load 不会**自动依赖 fetch 的 URL（避免凭据泄漏）——必须 `depends(url)` 显式声明。

## 15. untrack() 排除依赖

```js
// +page.js
export async function load({ untrack, url }) {
  // url.pathname 不再触发重跑
  if (untrack(() => url.pathname === '/')) {
    return { message: 'Welcome!' };
  }
}
```

## 16. getRequestEvent（共享 server 逻辑）

`src/lib/server/auth.js`

```js
import { redirect } from '@sveltejs/kit';
import { getRequestEvent } from '$app/server';

export function requireLogin() {
  const { locals, url } = getRequestEvent();
  if (!locals.user) {
    redirect(303, `/login?redirectTo=${encodeURIComponent(url.pathname + url.search)}`);
  }
  return locals.user;
}
```

`+page.server.js`：

```js
import { requireLogin } from '$lib/server/auth';

export function load() {
  const user = requireLogin();  // 已登录才会执行到这里
  return { message: `hello ${user.name}!` };
}
```

## 17. 认证布局模式（auth implications）

```js
// src/hooks.server.js —— 多个路由统一保护
export async function handle({ event, resolve }) {
  event.locals.user = await getUser(event.cookies.get('sessionid'));
  return resolve(event);
}

// 路由特定保护：直接放在 +page.server.js load 里（最直接、最稳）
export function load({ locals }) {
  if (!locals.user) redirect(303, '/login');
}

// 避免：把 auth guard 放 +layout.server.js（要求每个子页面 await parent）
```
