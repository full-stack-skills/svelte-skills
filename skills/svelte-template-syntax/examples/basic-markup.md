# Basic Markup (HTML++)

Svelte 组件模板是 "HTML++" — 与标准 HTML 几乎一致，但加入了表达式、属性简写、展开和事件处理。

## 1. 标签语义：小写=HTML，大写/点号=组件

```svelte
<script>
  import Widget from './Widget.svelte';
</script>

<!-- 小写标签 = HTML 元素 -->
<div>
  <span>text</span>
</div>

<!-- 大写或点号标签 = 组件 -->
<Widget />
<My.Component />
```

## 2. 静态属性

```svelte
<div class="foo">
  <button disabled>can't touch this</button>
</div>

<!-- 无引号属性值（与 HTML 兼容） -->
<input type=checkbox />
```

## 3. 表达式属性

```svelte
<a href="page/{p}">page {p}</a>

<!-- 整个属性值是表达式 -->
<button disabled={!clickable}>...</button>
```

## 4. 布尔属性：truthy/falsy 规则

```svelte
<!-- truthy → 包含属性；falsy → 排除 -->
<input required={false} placeholder="not required" />
<div title={null}>This div has no title attribute</div>
```

- **布尔属性**（`disabled`、`checked`、`required` 等）：truthy 包含，falsy 排除
- **其他属性**：仅当值为 `null` 或 `undefined` 时排除（其他 falsy 值会保留，例如 `0` 或 `''`）

## 5. 属性简写（name={name}）

```svelte
<script>
  let disabled = true;
</script>

<!-- 完整形式 -->
<button disabled={disabled}>...</button>

<!-- 简写形式 -->
<button {disabled}>...</button>
```

## 6. 展开属性

```svelte
<script>
  let things = $state({ a: 1, c: 3 });
</script>

<!-- 多个展开可与常规属性交错；后者覆盖前者 -->
<Widget a="b" {...things} c="d" />
<!-- 等价于：<Widget a={things.a ?? 'b'} {...things} c="d" /> -->
```

> 顺序决定优先级：先出现的属性被后出现的覆盖。

## 7. 文本表达式

```svelte
<script>
  let name = 'World';
  let a = 1, b = 2;
  let value = 'abc123';
</script>

<h1>Hello {name}!</h1>
<p>{a} + {b} = {a + b}.</p>

<!-- 表达式会被转义以防代码注入 -->
<div>{(/^[A-Za-z ]+$/).test(value) ? 'ok' : 'bad'}</div>
```

规则：
- `null` / `undefined` 会被省略
- 其他值（包括 `0`、`false`）会被强制转为字符串
- 正则字面量必须用括号包裹：`{(/.../) ...}`

## 8. 转义花括号

```svelte
<!-- 用 HTML 实体转义花括号 -->
<p>Use &lbrace; and &rbrace; to escape</p>
<p>或 &lcub; &rcub; 或 &#123; &#125;</p>
```

## 9. {@html} 原始 HTML

```svelte
<script>
  let rawHtml = '<strong>Bold</strong>';
</script>

{@html rawHtml}

<!-- ⚠️ XSS 风险：只对可信内容使用 -->
{@html sanitize(userInput)}
```

- 表达式被强制转为字符串后插入 DOM
- 不会转义，需确保内容可信
- 插入的内容不受 Svelte scoped 样式影响

## 10. 注释

```svelte
<!-- 标准 HTML 注释 -->
<h1>Hello</h1>

<!-- svelte-ignore 关闭警告 -->
<!-- svelte-ignore a11y_autofocus -->
<input bind:value={name} autofocus />

<!-- 组件文档注释（@component） -->
<!--
@component
- 描述：用户头像组件
- Usage:
  ```html
  <Avatar name="Ada" />
  ```
-->
```

## 11. 事件属性

```svelte
<!-- 事件属性以 on 开头（与属性规则一致） -->
<button onclick={() => console.log('clicked')}>click me</button>

<!-- 简写：name={name} 同样适用 -->
<button {onclick}>click me</button>

<!-- 展开形式 -->
<button {...thisSpreadContainsEvents}>click me</button>

<!-- 事件名区分大小写：onclick = click 事件，onClick = Click 事件 -->
```

## 12. 事件冒泡控制

```svelte
<div onclick={() => console.log('parent')}>
  <!-- 默认会冒泡到父级 -->
  <button onclick={() => console.log('child')}>Click</button>
</div>

<!-- 阻止冒泡 -->
<div onclick={() => console.log('parent')}>
  <button onclick={(e) => {
    e.stopPropagation();
    console.log('child only');
  }}>Click</button>
</div>
```

## 13. 事件委托（性能）

```svelte
<!-- 不推荐：每个按钮单独绑定 -->
{#each items as item}
  <button onclick={() => handle(item.id)}>{item.name}</button>
{/each}

<!-- 推荐：单监听器处理 -->
<ul onclick={(e) => {
  const target = e.target.closest('button');
  if (target) handle(Number(target.dataset.id));
}}>
  {#each items as item}
    <button data-id={item.id}>{item.name}</button>
  {/each}
</ul>
```

委托列表：`beforeinput`、`click`、`change`、`dblclick`、`contextmenu`、`focusin`、`focusout`、`input`、`keydown`、`keyup`、`mousedown`、`mousemove`、`mouseout`、`mouseover`、`mouseup`、`pointerdown`、`pointermove`、`pointerout`、`pointerover`、`pointerup`、`touchend`、`touchmove`、`touchstart`

## 14. 触摸事件默认为 passive

`ontouchstart` 和 `ontouchmove` 默认以 passive 模式注册以提升滚动性能。需 `preventDefault()` 时改用 `svelte/events` 的 `on(...)` 函数（在 action 中）。
