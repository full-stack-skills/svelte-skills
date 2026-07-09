# Reference: Page Options

完整 page options API，基于 `/tmp/sveltekit-llms.txt` 第 2089-2313 行。

## 公共规则

- 从 `+page.js` / `+page.server.js` / `+layout.js` / `+layout.server.js` 导出
- 子覆盖父
- 静态值（boolean / 字符串字面量）——SvelteKit 静态求值，**不导入**模块
- 动态值（如 `import { dev } from '$app/environment'; export const csr = dev;`）——SvelteKit 导入模块；浏览器代码**不能**在模块顶层运行

## prerender

```ts
export const prerender: boolean | 'auto' = true;
```

| 值 | 行为 |
| --- | --- |
| `true` | 构建时生成 HTML；不在 SSR manifest 中 |
| `false` | 仅 SSR；不在 prerender manifest 中 |
| `'auto'` | 可 prerender 也可 SSR；不在 SSR manifest 中 |

### 适用条件

- 任何两个用户直接访问都得相同内容
- 不能依赖 cookies / `url.searchParams` / 用户状态
- 有 form action 的页不能 prerender（POST 需 server）
- 动态参数路由可通过 `entries()` 列出可访问值

### Prerendering server routes

- `+server.js` 也支持 `prerender`
- 不受 layout 影响，但会从引用页继承默认值
- `+page.js` 设 `prerender = true` 且 `fetch('/my-server-route.json')` → 关联 `+server.js` 也视为可 prerender

### 命名冲突

```js
// 不可：foo + foo/bar 会冲突
src/routes/foo/+server.js
src/routes/foo/bar/+server.js

// 解决：加扩展名
src/routes/foo.json/+server.js
src/routes/foo/bar.json/+server.js
```

Pages 自动写 `foo/index.html` 而非 `foo` 绕开冲突。

### 报错

> "The following routes were marked as prerenderable, but were not prerendered"

修复方法：
1. 加 `entries()` 函数
2. 在 `svelte.config.js` 的 `kit.prerender.entries` 配置
3. 改 `prerender = 'auto'`

## entries

```ts
export function entries(): Array<Record<string, string>> | Promise<Array<Record<string, string>>>;
```

`/** @type {import('./$types').EntryGenerator} */` 提供类型。

可放在：`+page.js` / `+page.server.js` / `+server.js`（带动态参数的叶子，**不**是 layout）。

## ssr

```ts
export const ssr: boolean = true;
```

| 值 | 行为 |
| --- | --- |
| `true`（默认） | SSR + hydration |
| `false` | CSR（空 shell） |

> `ssr = false` + `csr = false` = 什么都不渲染，**禁止**。
> 根 layout 设 `ssr = false` → 整个 app 变 SPA。
> 不推荐用于 SSG（静态站点生成）。

## csr

```ts
export const csr: boolean = true;
```

| 值 | 行为 |
| --- | --- |
| `true`（默认） | Hydration + 客户端路由 |
| `false` | 无 JS，浏览器原生行为 |

`csr = false` 后果：
- `<script>` 块被剥掉
- `<form>` 不能用 `use:enhance`
- 链接变浏览器原生整页跳转
- HMR 失效

开发期保留 HMR：

```js
import { dev } from '$app/environment';
export const csr = dev;
```

## trailingSlash

```ts
export const trailingSlash: 'always' | 'never' | 'ignore' = 'never';
```

| 值 | 行为 |
| --- | --- |
| `'never'`（默认） | `/about/` → 301 → `/about` |
| `'always'` | 强制尾斜杠；prerender 输出 `about/index.html` |
| `'ignore'` | 不推荐——`/x` 和 `/x/` 是不同 URL |

可放在 `+layout.js` / `+layout.server.js` / `+server.js`。

## config

```ts
export const config: Record<string, unknown> = {
  // adapter-specific
};
```

- 对象**顶层 merge**（不深 merge）
- 适配器提供 `Config` interface，import 进来 typecheck
- 典型用法：指定 `runtime: 'edge'` / `regions: ['us1', 'us2']`

```ts
/** @type {import('some-adapter').Config} */
export const config = { runtime: 'edge' };
```

## PageProps / LayoutProps（Svelte 5 + SvelteKit 2.16+）

```svelte
<script>
  /** @type {import('./$types').PageProps} */
  let { data, form } = $props();
</script>
```

```svelte
<script>
  /** @type {import('./$types').LayoutProps} */
  let { data, children } = $props();
</script>
{@render children()}
```

Svelte 4 用 `export let data` + 单独类型声明。

## 详细参考链接

- [Page options (SvelteKit docs)](https://svelte.dev/docs/kit/page-options)
- [Adapters](https://svelte.dev/docs/kit/adapters)
- [Prerendering (SvelteKit docs)](https://svelte.dev/docs/kit/page-options#prerender)
