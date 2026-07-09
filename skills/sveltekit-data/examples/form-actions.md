# Examples: SvelteKit Form Actions

17 个常见 form action 场景，全部基于 `/tmp/sveltekit-llms.txt` 文档化模式。

## 1. Default action

`src/routes/login/+page.server.js`

```js
/** @satisfies {import('./$types').Actions} */
export const actions = {
  default: async (event) => {
    // TODO log the user in
  }
};
```

```svelte
<!-- +page.svelte -->
<form method="POST">
  <input name="email" type="email">
  <input name="password" type="password">
  <button>Log in</button>
</form>
```

无需 JS——浏览器原生 POST 到当前 URL。

## 2. Named actions

```js
export const actions = {
  login: async (event) => { /* ... */ },
  register: async (event) => { /* ... */ }
};
```

```svelte
<form method="POST" action="?/login">...</form>
<form method="POST" action="/login?/register">...</form>  <!-- 跨页 -->
```

> Default + named 不能共存——具名 action 不 redirect 时 `?/name` 留在 URL，下次 default POST 会命中它。

## 3. 同表单多 button（formaction）

```svelte
<form method="POST" action="?/login">
  <input name="email" type="email">
  <input name="password" type="password">
  <button>Login</button>
  <button formaction="?/register">Register</button>
</form>
```

## 4. 跨页调用（根 layout 的登录 widget）

```svelte
<!-- src/routes/+layout.svelte -->
<form method="POST" action="/login">
  <!-- POST 到 /login 的 default action -->
</form>
```

## 5. Anatomy：完整 action（cookies + 返回值）

```js
import * as db from '$lib/server/db';

export async function load({ cookies }) {
  const user = await db.getUserFromSession(cookies.get('sessionid'));
  return { user };
}

/** @satisfies {import('./$types').Actions} */
export const actions = {
  login: async ({ cookies, request }) => {
    const data = await request.formData();
    const email = data.get('email');
    const password = data.get('password');
    const user = await db.getUser(email);
    cookies.set('sessionid', await db.createSession(user), { path: '/' });
    return { success: true };
  },
  register: async (event) => { /* TODO */ }
};
```

```svelte
<script>
  /** @type {import('./$types').PageProps} */
  let { data, form } = $props();
</script>

{#if form?.success}
  <p>Welcome back, {data.user.name}!</p>
{/if}
```

返回值进 `form` prop——仅**本次响应**有效（reload 即消失）。

## 6. Validation errors（fail）

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

```svelte
<form method="POST" action="?/login">
  {#if form?.missing}<p class="error">Email is required</p>{/if}
  {#if form?.incorrect}<p class="error">Invalid credentials!</p>{/if}
  <input name="email" type="email" value={form?.email ?? ''}>
  <input name="password" type="password">
  <button>Log in</button>
  <button formaction="?/register">Register</button>
</form>
```

> **安全**：只 echo 允许的字段——**绝不**回显密码。

## 7. Redirect from action

```js
import { fail, redirect } from '@sveltejs/kit';

export const actions = {
  login: async ({ cookies, request, url }) => {
    const data = await request.formData();
    const email = data.get('email');
    const password = data.get('password');
    const user = await db.getUser(email);
    if (!user) return fail(400, { email, missing: true });
    if (user.password !== db.hash(password)) return fail(400, { email, incorrect: true });

    cookies.set('sessionid', await db.createSession(user), { path: '/' });

    if (url.searchParams.has('redirectTo')) {
      redirect(303, url.searchParams.get('redirectTo'));
    }
    return { success: true };
  }
};
```

## 8. Action 之后 load 重跑 + handle 不重跑

```js
// src/hooks.server.js
export async function handle({ event, resolve }) {
  event.locals.user = await getUser(event.cookies.get('sessionid'));
  return resolve(event);
}

// src/routes/account/+page.server.js
export function load(event) {
  return { user: event.locals.user };
}

export const actions = {
  logout: async (event) => {
    event.cookies.delete('sessionid', { path: '/' });
    event.locals.user = null;  // 必须手动更新！handle 不会重跑
  }
};
```

## 9. use:enhance（最小）

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

默认行为：避免整页刷新、更新 `form`/`page.form`/`page.status`（**仅**同页 action）、reset `<form>`、success 时 `invalidateAll`、redirect 时 `goto`、error 渲染 `+error.svelte`、重置焦点。

## 10. use:enhance 自定义（loading 状态 + applyAction）

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
        await applyAction(result);  // 等价默认 success/failure/error 行为
      }
    };
  }}
>
  <button disabled={submitting}>Save</button>
</form>
```

返回 callback 即覆盖默认 post-submit 行为，要恢复可用 `update()` 或 `applyAction(result)`。

## 11. 阻止提交（cancel）

```svelte
<form
  method="POST"
  use:enhance={({ cancel }) => {
    if (!confirm('Sure?')) cancel();
  }}
>
```

## 12. 完全手写（无 use:enhance）

```svelte
<script>
  import { invalidateAll, goto } from '$app/navigation';
  import { applyAction, deserialize } from '$app/forms';

  /** @param {SubmitEvent & { currentTarget: EventTarget & HTMLFormElement }} e */
  async function handleSubmit(e) {
    e.preventDefault();
    const data = new FormData(e.currentTarget, e.submitter);
    const response = await fetch(e.currentTarget.action, {
      method: 'POST',
      body: data
      // 如同路由有 +server.js，加：headers: { 'x-sveltekit-action': 'true' }
    });
    const result = deserialize(await response.text());

    if (result.type === 'success') {
      await invalidateAll();
    }
    applyAction(result);
  }
</script>
<form method="POST" onsubmit={handleSubmit}>...</form>
```

> 必须用 `deserialize`——不能 `JSON.parse`，result 含 `Date` / `BigInt`。

## 13. +server.js 与 action 同名冲突

```js
// src/routes/api/login/+server.js
export function POST() { /* 走这里 */ }

// src/routes/api/login/+page.server.js
export const actions = { default: async () => { /* 走这里 */ } };

// 前端：POST 到 action 必须加 header
const response = await fetch('/api/login', {
  method: 'POST',
  body: data,
  headers: { 'x-sveltekit-action': 'true' }
});
```

## 14. GET form（搜索/过滤，client-side router）

```svelte
<form action="/search">
  <input name="q">
  <button>Search</button>
</form>
```

不触发 action，按 client-side router 行为跳转到 `/search?q=...`。可用 `data-sveltekit-reload` / `data-sveltekit-replacestate` / `data-sveltekit-keepfocus` / `data-sveltekit-noscroll` 控制。

## 15. 文件上传（multipart）

`+page.server.js`：

```js
export const actions = {
  upload: async ({ request }) => {
    const data = await request.formData();
    const file = data.get('file');
    if (file instanceof File) {
      const buffer = Buffer.from(await file.arrayBuffer());
      await fs.writeFile(`uploads/${file.name}`, buffer);
    }
    return { success: true, name: file.name };
  }
};
```

```svelte
<form method="POST" action="?/upload" enctype="multipart/form-data">
  <input type="file" name="file">
  <button>Upload</button>
</form>
```

## 16. JSON API（+server.js 替代方案）

```svelte
<script>
  function rerun() {
    fetch('/api/ci', { method: 'POST' });
  }
</script>
<button onclick={rerun}>Rerun CI</button>
```

```js
// src/routes/api/ci/+server.js
export function POST() { /* do something */ }
```

Form actions 是首选（可渐进增强），但 `+server.js` 适合纯 JSON API。

## 17. 区分多 form 的回执

```js
export const actions = {
  save: async ({ request }) => {
    const data = await request.formData();
    // ...
    return { id: 'save', success: true };
  },
  delete: async ({ request }) => {
    const data = await request.formData();
    // ...
    return { id: 'delete', success: true };
  }
};
```

```svelte
{#if form?.id === 'save'}<p>Saved!</p>{/if}
{#if form?.id === 'delete'}<p>Deleted!</p>{/if}
```

> `form` prop 结构完全自定义；`/** @satisfies {import('./$types').Actions} */` 注解 + `ActionData` 类型保证类型安全。
