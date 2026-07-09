# Svelte Lifecycle Examples

本目录提供 Svelte 5 生命周期、Stores、Context、Testing 的可复制代码示例。

## 文件说明

| 文件 | 内容 |
|---|---|
| `lifecycle-hooks.md` | onMount 清理、onDestroy、tick DOM 测量、deprecated 钩子迁移 |
| `stores-advanced.md` | writable/readable/derived/自定义 store、异步流、cleanup |
| `store-patterns.md` | stores vs `$state`、localStorage 持久化、跨组件模式 |
| `context-advanced.md` | createContext、setContext、与 state 组合、SSR 安全 |
| `testing-vitest.md` | 单元/组件测试、$effect.root、flushSync、context wrapper |
| `testing-playwright.md` | e2e 测试、Playwright config、用户交互 |

## 使用建议

- 需要快速复制某段代码时查阅对应文件
- 需要深入原理/边界时参考 `../references/` 对应文档
- 与 `../SKILL.md` 的 Critical 章节配合使用

## 浏览器支持速查

所有示例假定运行在 Svelte 5 基线环境（Chrome 87+ / Firefox 83+ / Safari 14+）。涉及 `$state.snapshot` 等例外特性时单独标注。
