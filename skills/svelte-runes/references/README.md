# Svelte Runes References

本目录包含 Svelte 5 Runes 响应式系统的完整技术参考。

## 文件说明

| 文件 | 内容 |
|------|------|
| `runes-overview.md` | Runes 整体架构：vs Legacy 对比、生命周期、模块级 `$state` |
| `$state-deep.md` | `$state` 深层机制：Proxy 行为、`$state.raw`/`snapshot`/`eager`、内置响应式类、类中 `$state`、跨模块、getter 模式 |
| `$derived-deep.md` | `$derived` 深层机制：表达式 vs `$derived.by`、依赖追踪、可写派生、Push-pull 模型、解构 |
| `$effect-deep.md` | `$effect` 深层机制：lifecycle、cleanup、pre/tracking/pending/root、追踪规则、常见错误 |
| `$props-deep.md` | `$props` 完整参考：解构、Rest Props、Type safety、`$props.id()`、泛型、callback props |
| `$bindable-deep.md` | `$bindable` 完整参考：双向绑定、fallback、函数绑定组合、典型模式、SSR 行为 |
| `$inspect-deep.md` | `$inspect` 完整参考：`$inspect(...).with()`、`$inspect.trace()`、调试模式、与 `$effect` 对比 |
| `$host-deep.md` | `$host` 完整参考：自定义元素、`$host()` 派发事件、`<svelte:options>` 配置、典型模式 |
| `context-deep.md` | Context API：`createContext` vs `setContext`、类型安全、SSR 注意事项 |

## 深入学习路径

1. 先读 `runes-overview.md` 理解整体架构
2. 再按需查阅具体 Rune 的 deep 参考
3. 最佳实践见 `../examples/` 目录的可执行代码

## Rune 速查

- 状态/数据 → `$state-deep.md`
- 派生/计算 → `$derived-deep.md`
- 副作用 → `$effect-deep.md`
- 组件接口 → `$props-deep.md` + `$bindable-deep.md`
- 调试 → `$inspect-deep.md`
- 自定义元素 → `$host-deep.md`
- 跨组件共享 → `context-deep.md`
