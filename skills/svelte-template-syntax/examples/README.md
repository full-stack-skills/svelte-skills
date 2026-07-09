# Svelte Template Syntax Examples

本目录提供 Svelte 模板语法的离线可执行代码示例。

## 文件说明

| 文件 | 内容 |
|------|------|
| `basic-markup.md` | 标签、属性、展开、文本表达式、注释、事件委托、HTML 实体 |
| `if-await-snippet.md` | {#if}/{#each}/{#await}/{#snippet} 基础到高级用法 |
| `snippet-advanced.md` | Snippet 作用域、显式/隐式 prop、可选、类型化、模块导出、createRawSnippet、Snippet vs Slot |
| `render-tags.md` | {@render}/{@html}/{@debug}/{@const}/{@attach}、可选 snippet、scoped 样式 |
| `attachments.md` | {@attach} factory / inline / 条件 / 转换 action / createAttachmentKey |
| `use-action.md` | use: 指令 + Action 类型签名 + 从 action 迁移到 attachment |
| `transitions.md` | transition:/in:/out: 参数、内置列表、自定义函数 css/tick、crossfade、events |
| `animations.md` | animate:flip / 自定义函数 / 与 transition 组合 / 性能提示 |
| `await-expressions.md` | synchronized / 并发 / pending / $effect.pending / fork / SSR |
| `style-class.md` | style: 指令、class 对象/数组、class: 传统、ClassValue、!important |
| `bind-directives.md` | bind:value/checked/group/files/this/open + audio/video/img 绑定 + Function bindings |
| `event-handlers.md` | onclick vs on:click、事件冒泡、委托、window/document 事件 |

## 使用建议

- 需要复制模板代码时，直接从对应文件获取
- 需要理解语义差异时，结合 `references/` 目录的理论说明
- 完整模板语法参考见 `../SKILL.md`

## 主题对照表

| 主题 | 示例文件 |
|------|----------|
| 块语法 | if-await-snippet.md |
| 片段 | snippet-advanced.md, if-await-snippet.md, render-tags.md |
| 模板标签 | render-tags.md, attachments.md |
| 附件/动作 | attachments.md, use-action.md |
| 过渡/动画 | transitions.md, animations.md |
| 异步 | await-expressions.md, if-await-snippet.md |
| 绑定 | bind-directives.md |
| 事件 | event-handlers.md |
| 样式 | style-class.md |
| 基础 | basic-markup.md |
