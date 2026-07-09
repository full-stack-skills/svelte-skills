# Svelte Runes Examples

本目录提供 Svelte 5 Runes（响应式系统）的离线可执行代码示例。

## 文件说明

| 文件 | 内容 |
|------|------|
| `state-patterns.md` | `$state` 各种用法：基础、深层代理、类字段、跨模块 |
| `derived-patterns.md` | `$derived` 用法：基础、$derived.by、乐观 UI、解构 |
| `effect-patterns.md` | `$effect` 用法：基础、cleanup、pre、依赖追踪、禁忌 |
| `props-patterns.md` | `$props`/`$bindable` 用法：解构、类型、默认值、绑定 |

## 使用建议

- 需要快速复制 `$state`/`$derived`/`$effect` 代码时，查阅对应示例文件
- 需要理解 Runes 与 Svelte 4 的差异时，参考 `references/runes-vs-legacy.md`
- 需要深入原理时，结合 `../SKILL.md` 的 Critical 章节
