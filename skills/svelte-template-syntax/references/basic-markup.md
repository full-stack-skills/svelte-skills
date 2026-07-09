# Basic Markup Reference

Svelte 模板是 "HTML++" — 与标准 HTML 一致，加入表达式、属性简写、展开和事件。

## 标签语义

| 形式 | 含义 |
|------|------|
| `<div>`、`<span>` | HTML 元素（小写） |
| `<Widget>` | 组件（大写） |
| `<my.stuff>` | 组件（点号命名空间） |

```svelte
<script>
  import Widget from './Widget.svelte';
</script>

<div>
  <Widget />
</div>
```

## 属性规则

### 静态值

```svelte
<div class="foo">
  <button disabled>can't touch this</button>
</div>

<!-- 属性值可不加引号 -->
<input type=checkbox />
```

### 表达式值

```svelte
<!-- 字符串拼接 -->
<a href="page/{p}">page {p}</a>

<!-- 整个属性为表达式 -->
<button disabled={!clickable}>...</button>
```

### Truthy / Falsy 行为

- **布尔属性**（`disabled`、`checked` 等）：truthy 包含，falsy 排除
- **其他属性**：仅当 `null` 或 `undefined` 时排除

```svelte
<input required={false} placeholder="not required" />
<div title={null}>no title attribute</div>
```

> 引号包围单一表达式在 Svelte 6+ 会强制转为字符串 — Svelte 5 当前保留原类型。

### 简写

```svelte
<button {disabled}>...</button>
<!-- 等价 <button disabled={disabled}> -->
```

## 组件 props

```svelte
<Widget foo={bar} answer={42} text="hello" />
<!-- 简写同 name 时 -->
<Widget {foo} />
```

> 组件传入值称 **props**；元素传入值称 **attributes**。

## 展开属性

```svelte
<Widget a="b" {...things} c="d" />
<!-- 多个展开可与常规属性交错；后者覆盖前者 -->
```

## 事件属性

事件以 `on` 开头作为属性：

```svelte
<button onclick={() => console.log('clicked')}>click</button>
```

- 区分大小写：`onclick` = `click`，`onClick` = `Click`（自定义事件）
- 可用简写和展开
- 事件处理器在绑定更新**之后**触发

### 委托事件

为减少内存和提升性能，以下事件委托到根节点：

`beforeinput` `click` `change` `dblclick` `contextmenu` `focusin` `focusout` `input` `keydown` `keyup` `mousedown` `mousemove` `mouseout` `mouseover` `mouseup` `pointerdown` `pointermove` `pointerout` `pointerover` `pointerup` `touchend` `touchmove` `touchstart`

注意事项：
- 手动 `dispatchEvent` 需 `{ bubbles: true }` 才能到达根节点
- 手动 `addEventListener` 调 `stopPropagation` 会拦截委托
- 用 `svelte/events` 的 `on(...)` 函数保证顺序和 `stopPropagation` 正确

### 触摸事件

`ontouchstart` / `ontouchmove` 注册为 passive 以提升滚动性能。需 `preventDefault()` 时改用 `svelte/events` 的 `on(...)`。

## 文本表达式

```svelte
<p>{expression}</p>
```

- `null` / `undefined` 省略
- 其他值强制转字符串
- 表达式会被转义防注入

正则字面量必须用括号包裹：`{/(...)/.test(s) ? a : b}` → `{((/.../).test(s)) ? a : b}`。

### 转义花括号

```svelte
<p>Use &lbrace; and &rbrace;</p>
<!-- 或 &lcub; &rcub; 或 &#123; &#125; -->
```

### HTML 原始内容

```svelte
{@html potentiallyUnsafeHtmlString}
```

- 不转义；XSS 风险
- 不会编译 Svelte 代码
- 必须是**完整独立**的 HTML（不能跨 `{@html}` 标签拼接）
- **不受 scoped 样式影响** — 需 `:global`

## 注释

```svelte
<!-- 标准 HTML 注释 -->
<h1>Hello</h1>

<!-- svelte-ignore 关闭警告 -->
<!-- svelte-ignore a11y_autofocus -->
<input bind:value={name} autofocus />

<!-- 组件文档（@component） -->
<!--
@component
- 描述：用户头像
- Usage:
  ```html
  <Avatar name="Ada" />
  ```
-->
<script>
  let { name } = $props();
</script>

<main>
  <h1>Hello, {name}</h1>
</main>
```

## 元素 vs 组件解析规则

| 标签形式 | 解析为 |
|----------|--------|
| `<div>` | HTML 元素 |
| `<svelte:xxx>` | Svelte 特殊元素 |
| `<Capitalized>` | 已导入的组件 |
| `<lower.case>` | 组件（点号命名空间） |

## SSR 行为

- HTML 元素在 SSR 时正常输出
- 组件在 SSR 时渲染最终状态
- 事件、transitions、actions 在 SSR 阶段**不**运行
- 客户端 hydration 后才激活交互

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `on:click` 警告 | 已废弃 | 改用 `onclick` |
| `{@html}` 样式不生效 | 不受 scoped 影响 | 用 `:global` |
| 自定义事件大小写错 | 区分大小写 | 检查 onClick vs onclick |
| 事件不触发 | 委托事件未冒泡 | 用 `bubbles: true` 派发 |
