# Reference: When Load Functions Rerun

完整依赖追踪与手动失效文档，基于 `/tmp/sveltekit-llms.txt` 第 1349-1473 行。

## 核心机制

SvelteKit 跟踪每个 `load` 函数的依赖以避免重跑。**重跑 ≠ 重建组件**——`+layout.svelte` / `+page.svelte` 实例保留，只有 `data` prop 更新。

依赖追踪规则：
- 只对 `load` 函数**顶层 body** 生效
- 嵌套 promise 内访问的 `params.x` 不会被追踪（dev 警告）
- `request.url` 的属性**不**被追踪
- `url.searchParams.get/getAll/has` 单独追踪该参数

## 何时重跑（汇总）

| # | 条件 |
| --- | --- |
| 1 | 访问 `params` 某属性，值变了 |
| 2 | 访问 `url.pathname` / `url.search` 等，值变了 |
| 3 | `url.searchParams.get/getAll/has` 对应参数变了 |
| 4 | `await parent()` 且父 load 重跑了 |
| 5 | 子 load 调了 `await parent()` 且自己重跑 + 父是 server load |
| 6 | `fetch(url)` / `depends(url)` 声明依赖，且 `invalidate(url)` 被调 |
| 7 | `invalidateAll()` 被调 |

`params` / `url` 可在以下情况变化：
- `<a href="..">` 链接点击
- `<form>` 提交（GET 或 POST）
- `goto()` 调用
- `redirect()` 抛出

## invalidate()

```ts
// $app/navigation
function invalidate(dependency: string | ((url: URL) => boolean)): Promise<void>;
```

签名：
- `invalidate('app:random')` — 自定义 id（`[a-z]:` 前缀约定）
- `invalidate('https://api.example.com/random-number')` — URL 字符串
- `invalidate(url => url.href.includes('random-number'))` — 函数过滤
- 重跑所有声明了该依赖的 load 函数

**server load 不会**自动依赖 fetch 的 URL（避免凭据泄漏）——必须用 `depends(url)` 显式声明。

## invalidateAll()

```ts
function invalidateAll(): Promise<void>;
```

强制重跑**所有** active load 函数（server + universal）。

## depends()

```ts
// LoadEvent / ServerLoadEvent
function depends(...dependencies: string[]): void;
```

声明自定义依赖。约定前缀 `[a-z]:`：

```js
// +page.js
export async function load({ fetch, depends }) {
  depends('app:random');
  const response = await fetch('https://api.example.com/random-number');
  return { number: await response.json() };
}
```

```svelte
<script>
  import { invalidate } from '$app/navigation';
  function rerun() {
    invalidate('app:random');           // 触发 depends('app:random') 的 load 重跑
    invalidate('https://api.example.com/random-number');  // 触发 fetch 该 URL 的 load 重跑
  }
</script>
```

## untrack()

```ts
// LoadEvent / ServerLoadEvent
function untrack<T>(fn: () => T): T;
```

从依赖追踪中排除某段代码：

```js
export async function load({ untrack, url }) {
  // url.pathname 不再触发重跑
  if (untrack(() => url.pathname === '/')) {
    return { message: 'Welcome!' };
  }
}
```

## searchParams 追踪粒度

```js
export async function load({ url }) {
  const x = url.searchParams.get('x');  // 独立追踪 'x'
  const y = url.searchParams.get('y');  // 独立追踪 'y'
  const all = url.searchParams;          // 等同 url.search 整串追踪
}
```

`?x=1&y=1` → `?x=1&y=2` **不会**重跑只依赖 `y` 的 load。

## 组件重建

load 重跑**不**重建组件。需要强制重建用 `{#key}`：

```svelte
<script>
  import { page } from '$app/state';
  let { data } = $props();
</script>

{#key page.url.pathname}
  <BlogPost title={data.title} content={data.content} />
{/key}
```

或在 `afterNavigate` 回调里手动重置 state。

## 实用模式

### "刷新" 按钮

```svelte
<script>
  import { invalidateAll } from '$app/navigation';
</script>
<button onclick={invalidateAll}>Refresh</button>
```

### 提交后 invalidate

```svelte
<script>
  import { enhance } from '$app/forms';
  import { invalidate } from '$app/navigation';
</script>
<form method="POST" action="?/create" use:enhance={() => async ({ result }) => {
  if (result.type === 'success') invalidate('app:list');
}}>
  <!-- form fields -->
</form>
```

### 定时失效（轮询）

```svelte
<script>
  import { onMount, onDestroy } from 'svelte';
  import { invalidate } from '$app/navigation';
  let timer;
  onMount(() => {
    timer = setInterval(() => invalidate('app:status'), 5000);
  });
  onDestroy(() => clearInterval(timer));
</script>
```

## 认证场景

> Layout `load` 函数不在每次请求都跑（client-side navigation 之间复用）。Layout + page load 并发跑（除非 `await parent()`）。

**推荐**：
- 通用 auth 保护 → `hooks.server.js` 的 `handle`
- 路由特定保护 → `+page.server.js` load（最直接、最稳）
- **避免**：把 auth guard 放 `+layout.server.js`（要求所有子页面 `await parent()`）

## 详细参考链接

- [Rerunning load functions (SvelteKit docs)](https://svelte.dev/docs/kit/load#rerunning-load-functions)
- [$app/navigation](https://svelte.dev/docs/kit/$app-navigation)
