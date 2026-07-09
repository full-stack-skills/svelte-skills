# {@attach ...} — Attachments

`{@attach ...}` 是 Svelte 5.29+ 引入的声明式副作用机制，替代 `use:` action。attachment 是运行在 `$effect` 中的函数：元素挂载到 DOM 时调用，返回可选的清理函数（元素卸载或依赖变化时调用）。

## 1. 基本 attachment

```svelte
<script>
  /** @type {import('svelte/attachments').Attachment} */
  function myAttachment(element) {
    console.log(element.nodeName); // 'DIV'

    return () => {
      console.log('cleaning up');
    };
  }
</script>

<div {@attach myAttachment}>...</div>
```

一个元素可挂载多个 attachment。

## 2. Attachment Factory（带参）

```svelte
<script>
  import tippy from 'tippy.js';

  let content = $state('Hello!');

  /**
   * @param {string} content
   * @returns {import('svelte/attachments').Attachment}
   */
  function tooltip(content) {
    return (element) => {
      const t = tippy(element, { content });
      return t.destroy;
    };
  }
</script>

<input bind:value={content} />
<button {@attach tooltip(content)}>Hover me</button>
```

> 因为 `tooltip(content)` 表达式在 effect 中执行，`content` 变化时 attachment 会先销毁再重建。

## 3. 内联 attachment

```svelte
<canvas
  width={32}
  height={32}
  {@attach (canvas) => {
    const context = canvas.getContext('2d');

    $effect(() => {
      context.fillStyle = color;
      context.fillRect(0, 0, canvas.width, canvas.height);
    });
  }}
></canvas>
```

> 嵌套 effect 在 `color` 变化时执行；外层 effect（`getContext`）只运行一次。

## 4. 条件 attachment

```svelte
<div {@attach enabled && myAttachment}>...</div>
```

Falsy 值（`false`、`undefined`）被当作"无 attachment"。

## 5. 传给组件

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
<!-- App.svelte -->
<script>
  import tippy from 'tippy.js';
  import Button from './Button.svelte';

  function tooltip(content) {
    return (element) => {
      const t = tippy(element, { content });
      return t.destroy;
    };
  }
</script>

<Button {@attach tooltip('Save')}>Save</Button>
```

放在组件上的 attachment 会创建以 `Symbol` 为 key 的 prop；组件用展开 prop 时会传给根元素。

## 6. 控制 re-run 行为

`{@attach foo(bar)}` 会在 `foo` 或 `bar` 或 attachment 内部读取的状态变化时**完整重建**（与 action 不同）。

```js
// 每次 bar 变化都做昂贵的 setup
function foo(bar) {
  return (node) => {
    veryExpensiveSetupWork(node);
    update(node, bar);
  };
}

// 优化：把数据放进函数，在子 effect 中读取
function foo(getBar) {
  return (node) => {
    veryExpensiveSetupWork(node);

    $effect(() => {
      update(node, getBar());
    });
  };
}
```

## 7. 程序化创建 attachment

需要把 attachment 放到对象上时，用 `createAttachmentKey`：

```js
import { createAttachmentKey } from 'svelte/attachments';

const props = {
  [createAttachmentKey()]: (node) => {
    // ...挂载逻辑
    return () => { /* cleanup */ };
  }
};
```

## 8. 把 Action 转换为 Attachment

```js
// myAction.js (Svelte 4 use:action 形式)
export function myAction(node, params) {
  // ...
  return { destroy() { /* teardown */ } };
}
```

```svelte
<script>
  import { fromAction } from 'svelte/attachments';
  import { myAction } from './myAction.js';

  // 现在 myAction 可作为 attachment 使用
  const attach = fromAction(myAction);
</script>

<div {@attach attach({ ... })}>...</div>
```

## 9. 监听外部状态变化

```svelte
<script>
  import { onMount } from 'svelte';
  let visible = $state(true);
  let el;

  /** @type {import('svelte/attachments').Attachment} */
  function track(node) {
    const observer = new IntersectionObserver(([entry]) => {
      visible = entry.isIntersecting;
    });
    observer.observe(node);
    return () => observer.disconnect();
  }
</script>

<div bind:this={el} {@attach track}>
  {visible ? 'visible' : 'hidden'}
</div>
```

## 10. 多个 attachment 顺序

```svelte
<input
  {@attach (node) => { /* 1: focus */ node.focus(); }}
  {@attach (node) => { /* 2: validation */ validate(node); }}
  bind:value
/>
```

按出现顺序挂载，清理时反序。
