# Examples: SvelteKit Page Options

12 个常见 page options 场景，全部基于 `/tmp/sveltekit-llms.txt` 文档化模式。

## 1. prerender = true（基础）

`+page.js` 或 `+page.server.js`：

```js
export const prerender = true;
```

构建时生成静态 HTML。适用 marketing / docs / blog。

## 2. 根 layout 全开 + 个别页关闭

```js
// src/routes/+layout.js
export const prerender = true;
```

```js
// src/routes/dashboard/+page.js
export const prerender = false;  // dashboard 需要动态数据
```

子覆盖父——可在根开 prerender、个别页关闭。

## 3. prerender = 'auto'

```js
export const prerender = 'auto';
```

可 prerender 也可 SSR——`+server.js` 路由适合。

## 4. 动态路由用 entries() 列出可访问值

`src/routes/blog/[slug]/+page.server.js`

```js
/** @type {import('./$types').EntryGenerator} */
export function entries() {
  return [
    { slug: 'hello-world' },
    { slug: 'another-blog-post' }
  ];
}
export const prerender = true;
```

`entries` 可 `async`——从 CMS/DB 拉清单：

```js
export async function entries() {
  const posts = await db.getAllSlugs();
  return posts.map(p => ({ slug: p.slug }));
}
```

## 5. 配置 prerender 入口（svelte.config.js）

```js
// svelte.config.js
export default {
  kit: {
    prerender: {
      entries: ['*']  // 全部路由（默认）
    }
  }
};
```

适合告诉 prerender "这些 URL 要预渲染"——动态路由无 link 发现时必备。

## 6. Prerendering server routes

`+page.js`：

```js
export const prerender = true;

export async function load({ fetch }) {
  const res = await fetch('/my-server-route.json');
  return await res.json();
}
```

`src/routes/my-server-route.json/+server.js` 会被视为可 prerender（除非自己 `prerender = false`）。`+server.js` 不受 layout 影响，但会从引用页继承默认值。

## 7. ssr = false（SPA）

```js
// +page.js
export const ssr = false;
```

空 shell 页面。根 layout 设 `ssr = false` → 整个 app 变 SPA。

> `ssr = false` + `csr = false` = 什么都不渲染，**禁止**。

## 8. csr = false（无 JS）

```js
// +page.js
export const csr = false;
```

- `<script>` 被剥掉
- `<form>` 不能用 `use:enhance`
- 链接变浏览器原生整页跳转
- HMR 失效

## 9. csr = dev（开发期保留 HMR）

```js
// +page.js
import { dev } from '$app/environment';
export const csr = dev;
```

## 10. trailingSlash

```js
// src/routes/+layout.js
export const trailingSlash = 'always';
```

- `'never'`（默认）：`/about/` → 301 → `/about`
- `'always'`：所有链接强制尾斜杠；prerender 输出 `about/index.html` 而非 `about.html`
- `'ignore'`：**不推荐**——`/x` 和 `/x/` 是不同 URL，破坏 SEO

## 11. Route conflicts（命名冲突）

```js
// 不可：foo/+server.js + foo/bar/+server.js（生成 foo + foo/bar 冲突）
src/routes/foo/+server.js          // ❌
src/routes/foo/bar/+server.js      // ❌
```

解决：加扩展名

```js
src/routes/foo.json/+server.js     // ✅ foo.json
src/routes/foo/bar.json/+server.js // ✅ foo/bar.json
```

对于 pages：自动写 `foo/index.html` 而非 `foo`，绕开冲突。

## 12. config（adapter-specific）

```js
// +page.js
/** @type {import('some-adapter').Config} */
export const config = {
  runtime: 'edge'
};
```

对象**顶层 merge**（不深 merge）——子 layout / page 只覆盖需要改的 key：

```js
// +layout.js
export const config = {
  runtime: 'edge',
  regions: 'all',
  foo: { bar: true }
};

// +page.js —— 覆盖 regions 和 foo.baz
export const config = {
  regions: ['us1', 'us2'],
  foo: { baz: true }
};

// 结果：{ runtime: 'edge', regions: ['us1', 'us2'], foo: { baz: true } }
```

## Troubleshooting

**"marked as prerenderable, but were not prerendered"**：

```js
// 1. 加 entries()（动态路由）
/** @type {import('./$types').EntryGenerator} */
export function entries() {
  return [{ slug: 'a' }, { slug: 'b' }];
}

// 2. 加 svelte.config.js 的 kit.prerender.entries

// 3. 改 'auto'
export const prerender = 'auto';

// 4. 加 link 到该页（在另一个 prerenderable 页）
<a href="/blog/this-post">Read</a>
```
