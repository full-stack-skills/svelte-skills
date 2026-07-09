# {@attach} — Reference

Svelte 5.29+ 的声明式副作用机制，替代 `use:` action。Attachment 是运行在 `$effect` 中的函数，元素挂载到 DOM 时调用。

## 类型签名

```ts
type Attachment<Element = HTMLElement> = (
  element: Element
) => void | (() => void);
```

返回的函数在元素卸载或 attachment 重新运行时调用。

```svelte
<script>
  /** @type {import('svelte/attachments').Attachment} */
  function myAttachment(element) {
    // setup
    return () => {
      // cleanup
    };
  }
</script>

<div {@attach myAttachment}>...</div>
```

一个元素可有**多个** attachment。

## Attachment Factory

返回 attachment 的函数 — 用于参数化：

```svelte
<script>
  function tooltip(content) {
    return (element) => {
      const t = tippy(element, { content });
      return t.destroy;
    };
  }
</script>

<button {@attach tooltip(content)}>Hover me</button>
```

> 因为 `tooltip(content)` 在 effect 内求值，`content` 变化时 attachment 会**先销毁再重建**。

## 内联 attachment

```svelte
<canvas
  {@attach (canvas) => {
    const context = canvas.getContext('2d');

    $effect(() => {
      context.fillStyle = color;
      context.fillRect(0, 0, canvas.width, canvas.height);
    });
  }}
></canvas>
```

外层 effect 不读响应式 state → 只跑一次；嵌套 effect 读 `color` → 每次变化重跑。

## 条件 attachment

```svelte
<div {@attach enabled && myAttachment}>...</div>
<!-- 或 -->
<div {@attach condition ? attachA : attachB}>...</div>
```

Falsy 值（`false`、`undefined`）被视作无 attachment。

## 传给组件

放在组件上的 attachment 会创建以 `Symbol` 为 key 的 prop。组件展开 props 时会传到根元素：

```svelte
<!-- Button.svelte -->
<script>
  /** @type {import('svelte/elements').HTMLButtonAttributes} */
  let { children, ...props } = $props();
</script>

<button {...props}>
  {@render children?.()}
</button>
```

```svelte
<Button {@attach tooltip('Save')}>Save</Button>
```

## 响应式（vs action 关键区别）

`{@attach foo(bar)}` 会响应 `foo` 或 `bar` 变化（action 不会重新运行）：

```js
function foo(bar) {
  return (node) => {
    expensiveSetup(node);
    update(node, bar);
  };
}
```

优化：把数据放进函数，在子 effect 中读取：

```js
function foo(getBar) {
  return (node) => {
    expensiveSetup(node);

    $effect(() => {
      update(node, getBar());
    });
  };
}
```

## 程序化创建

把 attachment 放到对象上以便 spread：

```js
import { createAttachmentKey } from 'svelte/attachments';

const props = {
  [createAttachmentKey()]: (node) => {
    // mount logic
    return () => { /* cleanup */ };
  }
};
```

## 从 action 转换

`fromAction` 把 Svelte 4 形式的 action 包成 attachment：

```js
import { fromAction } from 'svelte/attachments';
import { myAction } from 'lib';

const attach = fromAction(myAction);
```

```svelte
<div {@attach attach(params)}>...</div>
```

## 何时使用

| 场景 | 推荐 |
|------|------|
| 第三方库挂载（tippy、IntersectionObserver、tooltip） | {@attach} |
| 包装组件透传行为 | {@attach}（通过展开） |
| 旧库只提供 use:action | {@attach fromAction(act)} |
| 简单且只运行一次 | 两者皆可 |
| 需要响应参数变化 | {@attach} |

## Action vs Attachment 详细对比

| 行为 | `use:` action | `{@attach}` |
|------|---------------|-------------|
| SSR | 不调用 | 不调用 |
| 参数变化 | 不重跑 | 重跑 |
| 用于组件 | 子元素不接收 | 通过 spread 接收 |
| 内部读响应式 state | 需 `$effect` | 自动响应 |
| 返回值 | `{ update?, destroy? }` 或 `$effect` 清理函数 | cleanup 函数 |
| 替代方案 | `fromAction(act)` 转换为 attachment | — |
| 删除后 | destroy 调用 | cleanup 调用 |

## 类型

```ts
// svelte/attachments
export type Attachment<Element = HTMLElement> = (
  element: Element
) => void | (() => void);

export function createAttachmentKey(): symbol;
export function fromAction<Action extends ActionReturn>(
  action: Action
): (node: Element, params: Parameters<Action>[1]) => void;
```

## 限制

- 不能用于 `<svelte:component>`、`<svelte:element>` 等动态元素（除非它们解析为具体组件）
- 同一元素多个 attachment 顺序固定：先声明先挂载，卸载反序
- 不能在 `bind:` 上挂 attachment（用 `bind:this` + 手动挂载）
- attachment 的参数在 effect 内求值 — 重算成本考虑

## 实际模板

```svelte
<!-- 单参数 -->
<div {@attach tooltip(text)}>{text}</div>

<!-- 多 attachment -->
<input
  bind:value
  {@attach autofocus}
  {@attach (node) => { node.select(); }}
/>

<!-- 包装组件 -->
<MyButton {@attach tooltip('Click me')}>Go</MyButton>

<!-- 条件 -->
<Modal {@attach shouldTrapFocus && trapFocus} />

<!-- 来自 action 库 -->
<div {@attach fromAction(legacyAction)({ x: 1 })}></div>
```
