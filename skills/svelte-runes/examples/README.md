# Svelte Runes Examples

本目录提供 Svelte 5 Runes（响应式系统）的离线可执行代码示例。

## 文件说明

| 文件 | 内容 |
|------|------|
| `state-patterns.md` | `$state` 各种用法：基础、深层代理、`$state.raw`/`snapshot`/`eager`、类字段、内置响应式类、跨模块共享、getter 模式 |
| `derived-patterns.md` | `$derived` 用法：基础、`$derived.by`、乐观 UI、解构派生、update propagation、async、push-pull |
| `effect-patterns.md` | `$effect` 用法：基础、cleanup、pre/tracking/pending/root、依赖追踪、untrack、禁忌场景 |
| `props-patterns.md` | `$props`/`$bindable` 用法：解构、类型安全、Rest Props、`$props.id()`、泛型组件、callback props |
| `bindable-patterns.md` | `$bindable` 全部用法：双向绑定、fallback、函数绑定、数组共享、验证写入 |
| `inspect-patterns.md` | `$inspect`/`$inspect.trace`/`$inspect.with` 调试模式与典型场景 |

## 使用建议

- 需要快速复制 `$state`/`$derived`/`$effect` 代码时，查阅对应示例文件
- 需要理解 Runes 与 Svelte 4 的差异时，参考 `../SKILL.md` 的 Critical 章节
- 需要深入原理时，结合 `../references/` 目录的 deep 参考
- 调试时使用 `inspect-patterns.md`；自定义元素时使用 `bindable-patterns.md` + `../references/$host-deep.md`
