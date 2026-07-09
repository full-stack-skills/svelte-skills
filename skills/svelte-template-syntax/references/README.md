# Svelte Template Syntax References

本目录包含 Svelte 模板语法的完整技术参考。

## 文件说明

| 文件 | 内容 |
|------|------|
| `basic-markup.md` | 标签语义、属性规则、事件委托、注释、SSR 行为、错误对照 |
| `if-await-each.md` | {#if}/{#else}/{#each}/{#await} 完整语法及 keyed 机制 |
| `snippet-render.md` | Snippet 定义/调用/递归/传参，@render/@const 类型签名 |
| `snippet-advanced.md` | 显式/隐式 prop、可选 snippet、createRawSnippet、模块导出、Snippet vs Slot |
| `special-tags.md` | {@html}/{@debug}/{@attach}/svelte:element/svelte:window/document/body |
| `attachments.md` | {@attach} 语义、reactive 重运行、fromAction 迁移、svelte/attachments API |
| `transitions.md` | 局部/全局、内置 transitions、自定义函数 css/tick、events、crossfade |
| `animations.md` | animate:flip、自定义函数（from/to DOMRect）、与 transition 组合、性能 |
| `await-expressions.md` | 同步更新、并发、$effect.pending()、settled()、fork、SSR |
| `style-class.md` | style: 优先级、class 对象/数组/ClassValue、!important 规则 |
| `bind-reference.md` | 所有 bind: 指令完整列表及用法 |
| `event-reference.md` | 事件处理：onclick vs on:click、window/document 事件 |

## 深入学习路径

1. 先读 `basic-markup.md` 和 `if-await-each.md` 理解块级语法基础
2. 再读 `snippet-render.md` 和 `snippet-advanced.md` 掌握 Snippet 系统
3. 接着 `bind-reference.md` 和 `event-reference.md` 处理数据流和交互
4. 进阶：`attachments.md`、`transitions.md`、`animations.md`、`await-expressions.md`、`style-class.md`
5. 结合 `examples/` 目录的代码示例加深理解

## 主题索引

| 主题 | 参考文件 |
|------|----------|
| 基础标记 | basic-markup.md |
| 块语法 | if-await-each.md |
| 片段 | snippet-render.md, snippet-advanced.md |
| 模板标签 | special-tags.md, attachments.md |
| 附件/动作 | attachments.md |
| 过渡/动画 | transitions.md, animations.md |
| 异步 | if-await-each.md, await-expressions.md |
| 绑定 | bind-reference.md |
| 事件 | event-reference.md |
| 样式 | style-class.md |
