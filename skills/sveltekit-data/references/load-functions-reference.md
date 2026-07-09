# Reference: Load Functions API

完整 `load` 函数 API 文档，基于 `/tmp/sveltekit-llms.txt` 第 776-1562 行。

## 文件位置

| 文件 | 类型 | 运行位置 |
| --- | --- | --- |
| `+page.js` | Universal page load | SSR + 浏览器 |
| `+page.server.js` | Server page load | 仅服务端 |
| `+layout.js` | Universal layout load | SSR + 浏览器 |
| `+layout.server.js` | Server layout load | 仅服务端 |

## LoadEvent（universal load）

传给 `+page.js` / `+layout.js` 的 `load` 函数：

```ts
interface LoadEvent {
  params: Record<string, string>;     // 路径参数
  route: { id: string | null };       // route id 如 '/blog/[slug]'
  url: URL;                           // 完整 URL（hash 在 SSR 不可用）
  data: unknown;                      // server load 返回值（仅当同时存在 server load）
  fetch: typeof fetch;                // 增强版 fetch
  setHeaders: (headers: Record<string, string>) => void;  // SSR 期间有效
  parent: () => Promise<unknown>;     // 父 load 的合并 data
  depends: (...deps: string[]) => void;  // 声明依赖
  untrack: <T>(fn: () => T) => T;     // 排除依赖追踪
}
```

## ServerLoadEvent（server load）

`+page.server.js` / `+layout.server.js` 的 `load` 函数继承 `RequestEvent`：

```ts
interface ServerLoadEvent extends LoadEvent {
  clientAddress: string;              // 客户端 IP
  cookies: Cookies;                   // 读写 cookies
  locals: App.Locals;                 // hooks 注入
  platform: App.Platform | undefined;  // adapter-specific
  request: Request;                   // 原始 Request
  setHeaders: (headers: Record<string, string>) => void;
}
```

## $types 模块

`./$types` 由 SvelteKit 自动生成：

```ts
// ./$types.d.ts
import type { PageLoad, PageServerLoad, LayoutLoad, LayoutServerLoad } from './$types';

export type PageLoad<OutputData = ...> = ...;
export type PageServerLoad = ...;
export type LayoutLoad = ...;
export type LayoutServerLoad = ...;
```

类型化 `load` 函数：

```js
/** @type {import('./$types').PageLoad} */
export function load({ params, fetch }) {
  return { post: ... };
}
```

页面接收 data：

```svelte
<script>
  /** @type {import('./$types').PageProps} */
  let { data } = $props();
  // data.post 自动推导
</script>
```

> SvelteKit 2.16.0+ 引入 `PageProps` / `LayoutProps` 简化类型声明。Svelte 4 用 `export let data` + 单独类型注解。

## Return value

### Universal load

- 任意 JS 值
- 包括 Svelte 组件 class / 函子实例 / class instance
- 不可序列化的内容**不**能跨 SSR → client 边界传递

### Server load

- 必须可被 [devalue](https://github.com/rich-harris/devalue) 序列化
- 支持：JSON 全部 + `BigInt` / `Date` / `Map` / `Set` / `RegExp` / 循环引用
- 不支持：class instance / Svelte 组件 / Symbol
- 自定义类型用 `transport` hook

### 合并规则

- 多个 `load` 函数（layout + page）返回值在运行时合并
- 同级 key 冲突时**后者覆盖**

## Cookies API（仅 server load / action）

```ts
interface Cookies {
  get(name: string, opts?: { path?: string }): string | undefined;
  getAll(opts?: { path?: string }): Array<{ name: string; value: string }>;
  set(name: string, value: string, opts: {
    path: string;           // 必填
    domain?: string;
    expires?: Date;
    httpOnly?: boolean;
    maxAge?: number;
    partitioned?: boolean;
    sameSite?: 'strict' | 'lax' | 'none';
    secure?: boolean;
  }): void;
  delete(name: string, opts?: { path?: string; domain?: string }): void;
  serialize(name: string, value: string, opts?: ...): string;
}
```

## fetch 增强（来自 SvelteKit）

- 继承 page request 的 `cookie` / `authorization` headers（仅同源/子域）
- 服务端支持相对 URL
- 内部 `+server.js` 调用直接走 handler
- SSR response 捕获并内联到 HTML
- hydration 时从 HTML 读出

```ts
type Fetch = typeof fetch;  // 签名同 Web fetch
```

## setHeaders 约束

- 只能设**未设置过**的 header
- 不同 load 函数设同 header 抛错
- 不能用 `setHeaders` 设 `set-cookie`（用 `cookies.set`）
- 浏览器端调用是 no-op

## parent() 行为

- 永远 `await parent()`
- `+page.js` / `+layout.js` 中 → 父 `+layout.js` 的 data
- `+page.server.js` / `+layout.server.js` 中 → 父 `+layout.server.js` 的 data
- 缺省 `+layout.js` 视为 `({ data }) => data`——自动桥接 server load 数据

## error / redirect

```ts
function error(status: number, body: { message: string } | string): never;
function redirect(status: 300..308, location: string): never;
```

- 都直接抛异常——SvelteKit 2.x 不再需要 `throw`（1.x 行为废弃）
- `redirect` 不要放在 `try` 块里——会被 catch 吃掉
- `error` 渲染最近 `+error.svelte`

## getRequestEvent

`$app/server` 导出，在 server load / action / hook 中可访问当前 `event`：

```js
import { getRequestEvent } from '$app/server';

export function requireLogin() {
  const { locals, url } = getRequestEvent();
  if (!locals.user) redirect(303, '/login');
  return locals.user;
}
```

## page.data（`$app/state`）

```ts
// $app/state
const page: {
  url: URL;
  params: Record<string, string>;
  route: { id: string | null };
  status: number;
  error: Error | null;
  data: App.PageData;        // 当前页面 load 的 data
  state: { [key: string]: any };
  form: ActionData | null;    // 当前 form action 的返回值
};
```

> SvelteKit < 2.12 或 Svelte 4 用 `$app/stores` 的 `$page`。

## 详细参考链接

- [Loading data (SvelteKit docs)](https://svelte.dev/docs/kit/load)
- [Errors (SvelteKit docs)](https://svelte.dev/docs/kit/errors)
- [$app/state](https://svelte.dev/docs/kit/$app-state)
