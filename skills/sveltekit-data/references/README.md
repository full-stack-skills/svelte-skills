# References

API-level 完整参考文档，可作为速查表使用。

| 文件 | 内容 |
| --- | --- |
| [`load-functions-reference.md`](./load-functions-reference.md) | `LoadEvent` / `ServerLoadEvent` / `$types` / return value 约束 / `Cookies` API / `fetch` 增强 / `setHeaders` 约束 / `parent()` / `error` / `redirect` / `getRequestEvent` / `page.data` |
| [`form-actions-reference.md`](./form-actions-reference.md) | `actions` 对象 / `RequestEvent` / `fail()` / `redirect()` / `error()` in actions / `form` prop / `use:enhance` 完整签名 / `SubmitFunction` / `applyAction` / `deserialize` / `x-sveltekit-action` header / hook 集成 / `ActionData` 类型 |
| [`page-options-reference.md`](./page-options-reference.md) | `prerender` (true/false/auto) / `entries` / `ssr` / `csr` / `trailingSlash` / `config` 全部选项 + 适用条件 + 约束 + 报错修复 / `PageProps` / `LayoutProps` |
| [`rerunning-loads-reference.md`](./rerunning-loads-reference.md) | 依赖追踪机制 / 何时重跑（7 种条件）/ `invalidate` / `invalidateAll` / `depends` / `untrack` / `searchParams` 追踪粒度 / 组件重建 / 实用模式 / 认证场景 |

每篇都基于 `/tmp/sveltekit-llms.txt` 原文整理，标注了源行号区间。
