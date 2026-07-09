---
name: sveltekit-data
description: SvelteKit 数据加载、表单、页面选项技能。当用户在 SvelteKit 中使用 +page.js/+page.server.js 加载数据、使用 +server.js API 路由、处理表单 actions（默认/具名/GET）、使用 use:enhance 进行渐进增强、配置 prerender/ssr/csr/trailingSlash 页面选项时使用。
---

# SvelteKit Data: Loading, Form Actions, Page Options (SvelteKit 2)

本技能覆盖 SvelteKit 数据层的三大支柱：(1) Loading data（`+page.js` / `+page.server.js` / `+layout.js` / `+layout.server.js` 的 `load` 函数、`page.data`、URL 数据、cookies/headers、错误与重定向、流式响应、依赖追踪与手动失效），(2) Form actions（默认/具名 actions、`fail` 验证、`use:enhance` 渐进增强、`GET` vs `POST`），(3) Page options（`prerender` / `entries` / `ssr` / `csr` / `trailingSlash` / `config`）。

## When to use this skill

- 决定 `+page.js` vs `+page.server.js` 的取舍（隐私凭据、序列化、组合使用）
- 在 `load` 函数中使用 `params` / `route` / `url` / `fetch` / `parent` / `depends` / `untrack`
- 用 `page.data` 在父布局访问子页面数据
- 用 `error(status, msg)` / `redirect(status, location)` 终止 load/action
- 用返回 Promise 实现 streaming 骨架屏
- 解决"load 函数何时重新执行"以及如何用 `invalidate` / `invalidateAll` / `untrack` 精确控制
- 在 server `load` / action 中读 `cookies` / 设响应头
- 设计 `+page.server.js` 的 actions：默认 action、具名 action、`?/name` 查询参数、`fail()` 验证
- 用 `use:enhance` 渐进增强表单、回填 `form` 字段、阻止默认重置、自定义 `applyAction`
- 决定哪些路由 `prerender = true` / `entries()` / `ssr = false` / `csr = false` / `trailingSlash`

## Critical: Loading data (universal vs server load functions)

SvelteKit 有两种 `load` 函数，运行位置和约束不同。

**Universal load**（`+page.js` / `+layout.js`）：默认 SSR 时在服务端跑一次、客户端 hydration 时再跑一次；之后所有导航都在浏览器内运行。可以返回**任意 JS 值**（包括 Svelte 组件构造函数等不可序列化的对象）。`fetch` 在 SSR 阶段内联到 HTML，hydration 时复用，不会泄漏私密凭据到客户端。

**Server load**（`+page.server.js` / `+layout.server.js`）：**永远在服务端跑**。返回值必须用 [devalue](https://github.com/rich-harris/devalue) 序列化（JSON + `BigInt` / `Date` / `Map` / `Set` / `RegExp` / 循环引用）。可访问 `cookies` / `locals` / `request` / `clientAddress` / `platform`。

```js
// +page.server.js
import * as db from '$lib/server/database';

/** @type {import('./$types').PageServerLoad} */
export async function load({ params }) {
  return { post: await db.getPost(params.slug) };
}
```

```js
// +page.js  ——  公共 API fetch，浏览器可直接拉
/** @type {import('./$types').PageLoad} */
export async function load({ fetch, params }) {
  const res = await fetch(`/api/items/${params.id}`);
  return { item: await res.json() };
}
```

**两者可以同时存在**。当同时存在时，server load 先跑，其返回值成为 universal load 的 `data` 参数；universal load 的返回值才到达页面：

```js
// +page.server.js —— 仅在服务端，返回 sessionId
export async function load({ locals }) {
  return { sessionId: locals.sessionId };
}

// +page.js —— 浏览器也能跑，但能拿到服务端上下文
export async function load({ data, fetch }) {
  const res = await fetch(`/api/me`, { headers: { 'x-session': data.sessionId } });
  return { me: await res.json() };
}
```

**何时用哪个**：
- 需要私密环境变量、数据库、文件系统 → server load
- 数据是公开 API、且希望减少服务器往返 → universal load
- 需要返回不可序列化对象（Svelte 组件 class） → universal load
- 需要 streaming（慢数据 + 骨架屏） → **必须 server load**（universal load 的 Promise 不会被流式传输）
- 需要在客户端也能重新跑 → universal load

## Critical: page.data (sharing data across components)

页面和所有祖先 layout 各有**自己的** `data` prop，包含它自己 + 全部祖先的合并数据。当**父**组件需要**子**组件返回的数据时，用 `$app/state` 的 `page.data`（`$app/stores` 的 `$page` 是旧式等价物）：

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import { page } from '$app/state';
  /** @type {import('./$types').LayoutProps} */
  let { data, children } = $props();
</script>

<svelte:head>
  <title>{page.data.title}</title>
</svelte:head>
{@render children()}
```

合并规则：同级出现相同 key 时**后者覆盖**。`+layout.js` 返回 `{ a:1, b:2 }` + `+page.js` 返回 `{ b:3, c:4 }` → `data = { a:1, b:3, c:4 }`。

## Critical: URL data (params/route/url)

`load` 函数通过 `url` / `route` / `params` 访问 URL。`url.hash` 在 SSR 阶段不可用（服务端无 fragment）。

```js
// src/routes/a/[b]/[...c]/+page.js
/** @type {import('./$types').PageLoad} */
export function load({ url, route, params }) {
  // route.id    = '/a/[b]/[...c]'
  // url         = URL 实例（origin/pathname/searchParams/...）
  // params.b    = 'x'   params.c = 'y/z'
  return { q: url.searchParams.get('q') };
}
```

`url.searchParams.get/getAll/has` 在依赖追踪中是**独立**的 key——`?x=1&y=1` → `?x=1&y=2` **不会**重跑只依赖 `y` 的 load。

## Critical: Cookies and Headers

**只有 server load** 可以读 / 写 `cookies`。`setHeaders` 在 universal load 中调用 SSR 时生效，浏览器内调用是 no-op。

```js
// +layout.server.js
export async function load({ cookies }) {
  const sessionid = cookies.get('sessionid');
  return { user: await db.getUser(sessionid) };
}

// +page.js —— 转发上游 cache 头
export async function load({ fetch, setHeaders }) {
  const response = await fetch('https://cms.example.com/products.json');
  setHeaders({ 'cache-control': response.headers.get('cache-control') });
  return response.json();
}
```

约束：`setHeaders` 同名 header 只能设一次；不能用 `setHeaders` 设 `set-cookie`（用 `cookies.set`）；`fetch` 只在**同源或子域**目标带 cookies，其他域用 `handleFetch` hook。

## Critical: Errors and Redirects

`error(status, message)` 和 `redirect(status, location)` 都**直接抛异常**——不要自己 `throw`（SvelteKit 1.x 行为已废弃）。`redirect` **不要放在 try 块里**（会被 catch 吃掉）。

```js
import { error, redirect } from '@sveltejs/kit';

export function load({ locals }) {
  if (!locals.user) error(401, 'not logged in');
  if (!locals.user.isAdmin) error(403, 'not an admin');
}

export function load({ locals, url }) {
  if (!locals.user) {
    const next = url.pathname + url.search;
    redirect(303, `/login?redirectTo=${encodeURIComponent(next)}`);
  }
}
```

`expected error`（用 `error()` 抛出）显示最近 `+error.svelte` 并带正确 status；`unexpected error` 触发 `handleError` hook，按 500 处理。

浏览器端导航用 `$app/navigation` 的 `goto`：

```js
import { goto } from '$app/navigation';
goto('/login');
```

## Critical: Streaming with promises

Server `load` 返回**未 await 的 Promise** 会被流式传输到浏览器，允许快数据先渲染、慢数据后到。

```js
// +page.server.js —— 关键数据先 await，慢数据后流
export async function load({ params }) {
  return {
    post: await loadPost(params.slug),
    comments: loadComments(params.slug)   // 不 await → 单独流
  };
}
```

```svelte
<!-- +page.svelte -->
<h1>{data.post.title}</h1>
{#await data.comments}
  <p>Loading comments...</p>
{:then comments}
  {#each comments as c}<p>{c.content}</p>{/each}
{:catch error}
  <p>error: {error.message}</p>
{/await}
```

**强约束**：
- streaming 仅在 JS 启用且非 Lambda/Firebase 等缓冲平台时生效
- 响应开始流式输出后**不能**再 `setHeaders` / `redirect`
- 手写 Promise 加 `.catch(() => {})` 防止 unhandled rejection；`fetch` 由 SvelteKit 自动处理
- 嵌套 promise 内的 `params.x` 不会被依赖追踪——必须在顶层 body 访问

## Critical: When load functions rerun

SvelteKit 跟踪每个 load 的依赖以避免重跑。`load` 重新执行的条件：

1. 访问 `params` 某属性且值变了
2. 访问 `url.pathname` / `url.search` 等且值变了
3. `url.searchParams.get/getAll/has` 对应参数变了
4. `await parent()` 且父 load 重跑了
5. 通过 `fetch(url)` 或 `depends(url)` 声明依赖，且 `invalidate(url)` 被调用
6. `invalidateAll()` 强制重跑所有 active load

```js
// +page.js —— 自定义依赖标签（约定 [a-z]: 前缀）
export async function load({ fetch, depends }) {
  depends('app:random');
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

**server load 不会**自动依赖 fetch 的 URL（避免泄漏凭据）——必须 `depends(url)` 显式声明。用 `untrack(fn)` 排除依赖：

```js
export async function load({ untrack, url }) {
  if (untrack(() => url.pathname === '/')) return { message: 'Welcome!' };
}
```

**重跑 ≠ 重建组件**。`+layout.svelte` / `+page.svelte` 实例保留，只有 `data` prop 更新，组件内部 state 保留。需要强制重建 → 用 `{#key page.url.pathname}`。

## Critical: Form actions (default/named, validation, redirects)

`+page.server.js` 导出 `actions` 对象提供 `<form>` 端点。**action 总是 POST**（GET 不应有副作用）。

**Default action**：

```js
// src/routes/login/+page.server.js
/** @satisfies {import('./$types').Actions} */
export const actions = {
  default: async (event) => { /* ... */ }
};
```

```svelte
<form method="POST">
  <input name="email">
  <input name="password" type="password">
  <button>Log in</button>
</form>
```

**Named actions**：用 `?/name` 区分：

```js
export const actions = {
  login: async (event) => { /* ... */ },
  register: async (event) => { /* ... */ }
};
```

```svelte
<form method="POST" action="?/login">...</form>
<form method="POST" action="/login?/register">...</form>  <!-- 跨页调用 -->
<!-- 同表单不同按钮 -->
<form method="POST" action="?/login">
  <button>Login</button>
  <button formaction="?/register">Register</button>
</form>
```

**重要**：default + named 不能共存——若 POST 具名 action 不 redirect，`?/name` 留在 URL 里，下次 default POST 也会命中它。

**Validation errors**：`fail(status, data)` 返回 status + 数据，data 进 `form` prop / `page.form` / `page.status`：

```js
import { fail } from '@sveltejs/kit';
export const actions = {
  login: async ({ cookies, request }) => {
    const data = await request.formData();
    const email = data.get('email');
    const password = data.get('password');
    if (!email) return fail(400, { email, missing: true });
    const user = await db.getUser(email);
    if (!user || user.password !== db.hash(password)) {
      return fail(400, { email, incorrect: true });
    }
    cookies.set('sessionid', await db.createSession(user), { path: '/' });
    return { success: true };
  }
};
```

**Anatomy**：action 接收 `RequestEvent`，可读 `request.formData()`，可写 `cookies`，可 return / `fail` / `redirect` / `error`。返回值进 `form` prop，仅**本次响应**有效（reload 即消失）。**回填安全**——只 echo 用户允许的字段，**绝不**回显密码。

**Redirects**：和 load 一样用 `redirect(status, location)`。

**Action 之后的 `load`**：action 完成后（除非 redirect / unexpected error）页面会重渲染——`load` 函数会重跑。`handle` hook **不会**重跑——若你在 `handle` 里从 cookie 读 `locals.user`，action 修改 cookie 后必须**手动更新** `event.locals`。

## Critical: use:enhance (progressive enhancement)

`use:enhance` 是 `<form>` 的 action，给表单加上"无 JS 也工作、有 JS 时更平滑"的能力。**只**用于 `method="POST"` + `+page.server.js` action；用于 `+server.js` 或 GET 都会报错。

**最小用法**：

```svelte
<script>
  import { enhance } from '$app/forms';
  /** @type {import('./$types').PageProps} */
  let { form } = $props();
</script>
<form method="POST" use:enhance>
  <input name="email" value={form?.email ?? ''}>
</form>
```

**默认行为**：模拟浏览器原生但避免整页刷新——更新 `form` / `page.form` / `page.status`（**仅**同页 action，否则不会更新）、reset `<form>`、success 时 `invalidateAll`、redirect 时 `goto`、error 时渲染最近 `+error.svelte`、重置焦点。

**自定义 SubmitFunction**（返回 callback 即覆盖默认 post-submit 行为，要恢复可用 `update()` 或 `applyAction(result)`）：

```svelte
<script>
  import { enhance, applyAction } from '$app/forms';
  import { goto } from '$app/navigation';
  let submitting = $state(false);

  /** @type {import('./$types').PageProps} */
  let { form } = $props();
</script>

<form
  method="POST"
  use:enhance={({ formElement, formData, action, cancel, submitter }) => {
    submitting = true;
    return async ({ result, update }) => {
      submitting = false;
      if (result.type === 'redirect') {
        goto(result.location, { invalidateAll: true });
      } else {
        await applyAction(result);  // 等价于默认 success/failure/redirect/error 行为
      }
    };
  }}
>
  <button disabled={submitting}>Save</button>
</form>
```

**完全手写**（无 `use:enhance`）：用 `submit` 事件 + `fetch` + `deserialize`（**不能**用 `JSON.parse`，因为 result 含 `Date` / `BigInt`）：

```svelte
<script>
  import { invalidateAll, goto } from '$app/navigation';
  import { applyAction, deserialize } from '$app/forms';
  /** @param {SubmitEvent & { currentTarget: EventTarget & HTMLFormElement }} e */
  async function handleSubmit(e) {
    e.preventDefault();
    const data = new FormData(e.currentTarget, e.submitter);
    const response = await fetch(e.currentTarget.action, {
      method: 'POST', body: data,
      headers: { 'x-sveltekit-action': 'true' }  // 同名 +server.js 时强制走 action
    });
    const result = deserialize(await response.text());
    if (result.type === 'success') await invalidateAll();
    applyAction(result);
  }
</script>
<form method="POST" onsubmit={handleSubmit}>...</form>
```

**同路由有 `+server.js` 时**，`fetch` 默认走 `+server.js`。要强制 POST 到 action 必须加 header `x-sveltekit-action: true`。

## Critical: Page options (prerender/entries/ssr/csr/trailingSlash)

Page options 控制整页（或子树）的渲染方式。从 `+page.js` / `+page.server.js` / `+layout.{js,server.js}` 导出。**子覆盖父**——可在根 layout 开 prerender、个别页关闭。

```js
// +page.js / +layout.js / +page.server.js
export const prerender = true;          // 构建时生成 HTML
export const prerender = false;         // 显式禁用（用于根 layout 开启全部 prerender 的场景）
export const prerender = 'auto';        // 可 prerender 也可 SSR（不写入 manifest 排除）
export const ssr = false;               // 仅 CSR——空 shell
export const csr = false;               // 不发任何 JS
export const trailingSlash = 'always' | 'never' | 'ignore';

// entries —— 动态路由告诉 prerender 哪些值
// src/routes/blog/[slug]/+page.server.js
/** @type {import('./$types').EntryGenerator} */
export function entries() {
  return [{ slug: 'hello-world' }, { slug: 'another-post' }];  // 可 async
}

// config —— adapter-specific
/** @type {import('some-adapter').Config} */
export const config = { runtime: 'edge' };
```

- `prerender`：内容对所有用户相同（marketing/docs/blog）；不适用 cookies/`url.searchParams`/用户状态/form action（POST 需 server）。动态路由用 `entries()` 或 `kit.prerender.entries`。报错 "marked as prerenderable, but were not prerendered" → 加 `entries` / link / 改 `'auto'`
- `ssr = false`：根 layout 设整个 app 变 SPA。`ssr = false` + `csr = false` = 什么都不渲染，**禁止**
- `csr = false`：`<script>` 被剥掉，`<form>` 不可用 `use:enhance`，链接变浏览器原生跳转，HMR 失效。开发期 `csr = dev;` 保留 HMR
- `trailingSlash`：`'never'`（默认）`/about/` → 301 → `/about`；`'always'` prerender 输出 `about/index.html`；`'ignore'` 不推荐（破坏 SEO）
- `config` 对象**顶层 merge**（不深 merge）——子 layout / page 只覆盖需要改的 key

## Quick Fixes

- 私密 API 暴露凭据 → 改 `+page.server.js` 或 universal 中转不敏感字段
- `data` prop `undefined` → 漏 `let { data } = $props();` 或类型声明
- 流式数据 hydration 丢失 → universal load 的 Promise 不会流传输，改 `+page.server.js`
- streaming 中 `setHeaders` 报错 → 响应开始流后 header 不可改
- `fail()` 后 form 字段空 → 第二参必须含回显字段（`{ email }`）+ `value={form?.email ?? ''}`
- action 改 cookie 后页面还是旧用户 → `handle` 只跑一次，必须在 action 里手动 `event.locals.user = ...`
- `use:enhance` 不生效 → 必须是 `method="POST"` + POST 到 `+page.server.js` action
- "marked as prerenderable, but not prerendered" → 加 `entries()` 或 `kit.prerender.entries`
- `entries()` 位置错 → 必须在**带动态参数的叶子**（`+page.js`/`+page.server.js`/`+server.js`），不是 layout
- `parent()` 瀑布 → 不依赖 `parent()` 的 `getData(params)` 先 await 再 `await parent()`
- `url.hash` SSR 阶段 undefined → 服务端无 fragment；改 `onMount` 读

## Gotchas

- **Server load 返回值必须可序列化**：`Map`/`Set`/`Date`/`BigInt`/`RegExp`/循环引用 OK（devalue），但**不能**返回 Svelte 组件 class / class instance（除非 transport hook 自定义）
- **load 函数应纯净**——不要在 `+page.server.js` 顶层 `let user` 跨请求共享（多租户长生命周期，状态会泄漏）
- **依赖追踪只对顶层 body 生效**：await 后的 promise 内访问 `params.x` 不会触发重跑（dev 警告）
- **searchParams 追踪粒度**：`get`/`getAll`/`has` 独立；`url.searchParams` 整体访问等同 `url.search` 整串追踪
- **`form` prop 仅响应存在**——刷新即清空（仅本次提交回执，非持久数据）
- **`use:enhance` 默认不更新跨页 `form`**：从 `/a` POST `/b?action` 时 `/a` 的 form 不更新——需 `applyAction(result)`
- **`+server.js` 与 `+page.server.js` 同名冲突**：`fetch('/x')` 默认走 `+server.js`；POST 到 action 必须加 `x-sveltekit-action: true` header
- **`+page.js` 桥接 server load**：缺省 layout.js 视为 `({ data }) => data`，自动传 server load 数据
- **streaming + 重定向冲突**：响应开始流后 header 不可改；不能在流出的 promise 内 `redirect`
- **Lambda/Firebase 缓冲整页**——不持流式响应
- **`trailingSlash: 'ignore'` 破坏 SEO**：`/x` 和 `/x/` 是不同 URL
- **`csr = false` 与 HMR 不兼容**：开发期 `csr = dev;` 临时开启
- **`redirect` 在 `try {...}` 中被 catch**——直接 `redirect()` 不要包 try
- **回显安全**：form 数据不要 echo 密码/token，仅回显允许的字段

## FAQ

**Q: `+page.js` vs `+page.server.js` 选哪个？** A: 私密凭据/DB/cookies → server；公共 API/不可序列化对象 → universal。两者可同时存在，server 先跑。

**Q: `page.data` vs `data` prop？** A: `data` prop = 当前组件 + 全部祖先的合并；`page.data` = 当前页面返回的数据，从任意祖先可读。

**Q: 怎么让 load 强制重跑？** A: `invalidate(url)` 精确失效（按 URL 或 `depends` 标签），或 `invalidateAll()` 全量。server load 中 `fetch(url)` 不会自动依赖 URL，必须 `depends(url)`。

**Q: form 提交后 `form` prop 没了？** A: 正常。`form` prop 只在响应存在时存在；刷新即清空。持久数据应入 DB 后由 `load` 读。

**Q: `use:enhance` + redirect？** A: 默认就调 `goto(result.location)`。自定义可在 callback 里判 `result.type === 'redirect'` 后 `goto(result.location, { invalidateAll: true })`。

**Q: prerender `/blog/[slug]` 动态路由？** A: 加 `entries()` 函数返回 slug 列表，或在 `svelte.config.js` 的 `kit.prerender.entries` 配置。

**Q: 整个 app 变 SPA？** A: 根 `+layout.js` 设 `export const ssr = false;`（不推荐 SSG）。

**Q: 禁用 JS？** A: `export const csr = false;`——无 hydration、`<form>` 仍工作（POST 整页刷新）。

**Q: 同表单提交到不同 action？** A: `<button formaction="?/other">` 覆盖 `<form action>`。

**Q: action 数据类型安全？** A: `/** @satisfies {import('./$types').Actions} */` 注解；`form` prop 用 `ActionData` 类型。

**Q: streaming 在 serverless 能用吗？** A: Lambda/Firebase 缓冲整页；NGINX 需配置不缓冲。

**Q: `parent()` 同步还是异步？** A: 必须 `await parent()`。

## Examples & References tables

### 关键 API 一览

| 概念 | 关键文件 / API |
| --- | --- |
| Universal load | `+page.js` / `+layout.js` `PageLoad` / `LayoutLoad` |
| Server load | `+page.server.js` / `+layout.server.js` `PageServerLoad` / `LayoutServerLoad` |
| 跨组件读数据 | `$app/state` 的 `page` / `App.PageData` |
| URL 数据 | `params` / `route.id` / `url`（hash SSR 不可用） |
| Cookies / Headers | `cookies` / `setHeaders`（仅 server） |
| 错误/重定向 | `error()` / `redirect()` / `goto` |
| Streaming | server load 返回未 await 的 Promise |
| 依赖追踪 | `depends` / `untrack` / `invalidate` / `invalidateAll` |
| Form actions | `actions` / `fail` / `redirect` / `form` prop |
| Progressive enhancement | `enhance` / `applyAction` / `deserialize` |
| Page options | `prerender` / `entries` / `ssr` / `csr` / `trailingSlash` / `config` |

### 文件索引

| 文件 | 主题 | 示例数 |
| --- | --- | --- |
| `examples/load-functions.md` | load 函数全谱 | 17 |
| `examples/form-actions.md` | form action + use:enhance | 17 |
| `examples/page-options.md` | page options 全谱 | 12 |

| 文件 | 主题 |
| --- | --- |
| `references/load-functions-reference.md` | `load` 完整 API / 参数 / 返回值 / `$types` |
| `references/form-actions-reference.md` | action API / `use:enhance` 回调 / hook 集成 |
| `references/page-options-reference.md` | 全部 page options / 约束 / 行为 |
| `references/rerunning-loads-reference.md` | 何时重跑 / 手动 invalidate / untrack |
