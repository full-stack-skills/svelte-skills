# Svelte Runes References

本目录包含 Svelte 5 Runes 响应式系统的完整技术参考。

## 文件说明

| 文件 | 内容 |
|------|------|
| `runes-overview.md` | Runes 整体架构：vs Legacy 对比、生命周期、模块级 $state |
| `$state-deep.md` | $state 深层机制：Proxy 行为、$state.raw、$state.snapshot、$state.eager |
| `$derived-deep.md` | $derived 深层机制：表达式 vs $derived.by、依赖追踪、解构、自动重算 |
| `$effect-deep.md` | $effect 深层机制：tracking/pre/cleanup、$effect.root、禁忌场景 |
| `$props-deep.md` | $props 完整参考：解构、$bindable、Rest Props、类型安全 |
| `context-deep.md` | Context API：createContext vs setContext、类型安全、SSR 注意事项 |

## 深入学习路径

1. 先读 `runes-overview.md` 理解整体架构
2. 再按需查阅具体 Rune 的 deep 参考
3. 最佳实践见 `examples/` 目录的可执行代码
