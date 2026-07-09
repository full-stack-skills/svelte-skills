# Reference: Form Actions & use:enhance

完整 form actions API，基于 `/tmp/sveltekit-llms.txt` 第 1563-2087 行。

## actions 对象（+page.server.js）

```ts
/** @satisfies {import('./$types').Actions} */
export const actions: Actions = {
  default: Action,
  named: Action
};

interface Action {
  (event: RequestEvent): Promise<{
    success: true;
    [key: string]: any;       // 自定义返回字段
  } | ActionFailure> | ...;
}
```

- `default` / 具名 actions
- 总是 `POST`（GET 不应有副作用）
- 不能 default + named 共存——具名 action 不 redirect 时 `?/name` 留在 URL，下次 default POST 命中它

## RequestEvent（action 接收）

同 server load + form 特有：

```ts
interface RequestEvent {
  request: Request;        // POST 请求
  url: URL;
  params: Record<string, string>;
  route: { id: string | null };
  cookies: Cookies;
  locals: App.Locals;
  fetch: typeof fetch;
  setHeaders: (headers: Record<string, string>) => void;
  // ...
}
```

## fail()

```ts
function fail<T = Record<string, unknown>>(
  status: number,
  data?: T
): ActionFailure<T>;

interface ActionFailure<T> {
  status: number;          // 4xx/5xx
  data: T;                 // 进 form prop
}
```

- status 进 `page.status`
- data 进 `form` prop / `page.form`
- 推荐 400（客户端错）或 422（验证错）

## redirect() / error() in actions

```ts
import { redirect, error } from '@sveltejs/kit';

export const actions = {
  login: async ({ cookies, request, url }) => {
    // ...
    if (!user) return fail(400, { email, missing: true });
    if (url.searchParams.has('redirectTo')) {
      redirect(303, url.searchParams.get('redirectTo'));
    }
    return { success: true };
  }
};
```

## form prop

```ts
// +page.svelte
let { form }: { form: ActionData | null } = $props();
// form = { success: true, ... } | { missing: true, email } | null
```

- 仅响应存在（reload 即清空）
- 跨页 action 不更新当前页 form（除非 `applyAction(result)`）
- `ActionData` 类型由 `/** @satisfies ... */` 推导

## use:enhance

```ts
// $app/forms
export function enhance(form_element: HTMLFormElement, submit?: SubmitFunction): {
  destroy(): void;
};

type SubmitFunction = (opts: {
  formElement: HTMLFormElement;
  formData: FormData;
  action: URL;
  cancel: () => void;
  submitter: HTMLElement | null;
  controller: AbortController;
}) => void | ((opts: { result: ActionResult; update: (opts?: { reset?: boolean; invalidateAll?: boolean }) => Promise<void> }) => Promise<void>);
```

### 默认行为（无 callback）

```svelte
<form method="POST" use:enhance>
```

- success/failure：更新 `form` / `page.form` / `page.status`（**仅**同页 action），reset `<form>`，invalidateAll
- redirect：goto
- error：渲染最近 `+error.svelte`，重置焦点

### 自定义 SubmitFunction

```svelte
<form
  method="POST"
  use:enhance={({ formElement, formData, action, cancel, submitter, controller }) => {
    // 提交前逻辑
    if (!confirm('Sure?')) cancel();
    return async ({ result, update }) => {
      // 提交后逻辑
      if (result.type === 'redirect') {
        goto(result.location, { invalidateAll: true });
      } else {
        await applyAction(result);
      }
    };
  }}
>
```

返回 callback 即覆盖默认 post-submit 行为。要恢复默认：
- `update({ reset: boolean, invalidateAll: boolean })`
- `applyAction(result)`（按 result.type 分发）

### cancel()

阻止表单提交。`controller.abort()` 取消 inflight 请求。

## applyAction()

```ts
function applyAction(result: ActionResult): void;

interface ActionResult {
  type: 'success' | 'failure' | 'redirect' | 'error';
  status: number;
  // success / failure 才有 data
  data?: any;
  // redirect 才有 location
  location?: string;
  // error 才有 error
  error?: any;
}
```

`applyAction(result)` 行为：
- `success` / `failure`：设 `page.status`、更新 `form` / `page.form`（**无论**从哪提交）
- `redirect`：`goto(result.location, { invalidateAll: true })`
- `error`：渲染最近 `+error.svelte` with `result.error`
- 全部：重置焦点

## deserialize()

```ts
function deserialize<T = ActionResult>(data: string): T;
```

服务端返回 devalue 序列化的字符串，**必须**用 `deserialize` 反序列化（不能 `JSON.parse`）——result 含 `Date` / `BigInt` / 循环引用。

## x-sveltekit-action header

当同路由有 `+server.js` 时，`fetch` 默认走 `+server.js`。要强制 POST 到 action：

```js
fetch(this.action, {
  method: 'POST',
  body: data,
  headers: { 'x-sveltekit-action': 'true' }
});
```

## ActionData / ActionFailure 类型

```ts
// ./$types
import type { ActionData, ActionFailure } from './$types';

// ActionData = 联合所有 action 成功返回值
// ActionFailure<T> = { status, data: T }
```

## Hook 集成（handle + action）

```js
// src/hooks.server.js
export async function handle({ event, resolve }) {
  event.locals.user = await getUser(event.cookies.get('sessionid'));
  return resolve(event);
}
```

> `handle` 在 action 之前跑；**action 后不重跑**。若 action 改 cookie，必须手动 `event.locals.user = ...`。

## GET form（无 action）

```svelte
<form action="/search">
  <input name="q">
</form>
```

- 不触发 action
- 按 client-side router 行为跳转到 `/search?q=...`
- 可用 `data-sveltekit-reload` / `data-sveltekit-replacestate` / `data-sveltekit-keepfocus` / `data-sveltekit-noscroll`

## 详细参考链接

- [Form actions (SvelteKit docs)](https://svelte.dev/docs/kit/form-actions)
- [$app/forms](https://svelte.dev/docs/kit/$app-forms)
- [Progressive enhancement](https://svelte.dev/docs/kit/form-actions#progressive-enhancement)
