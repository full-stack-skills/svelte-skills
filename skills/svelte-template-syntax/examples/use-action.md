# use: Action

Actions 是元素挂载时调用的函数。Svelte 5.29+ 推荐改用 {@attach}，但 `use:` 仍受支持。

## 1. 基本 action

```svelte
<script>
  /** @type {import('svelte/action').Action} */
  function myaction(node) {
    // node 已挂载到 DOM

    $effect(() => {
      // setup
      return () => {
        // teardown
      };
    });
  }
</script>

<div use:myaction>...</div>
```

## 2. 带参数的 action

```svelte
<script>
  /** @type {import('svelte/action').Action<HTMLElement, { color: string }>} */
  function paint(node, data) {
    node.style.color = data.color;
  }
</script>

<div use: paint={{ color: 'red' }}>red text</div>
```

> action 只调用一次（不进行 SSR）；参数变化**不会**重新调用。

## 3. Action 类型签名

```svelte
<script>
  import type { Action } from 'svelte/action';

  /**
   * @type {Action<
   *   HTMLDivElement,
   *   { color: string } | undefined,
   *   { onswipeleft: (e: CustomEvent) => void; onswiperight: ... }
   * >}
   */
  function gestures(node, params) {
    $effect(() => {
      // ...
      node.dispatchEvent(new CustomEvent('swipeleft'));
    });
  }
</script>

<div
  use:gestures={{ color: 'red' }}
  onswipeleft={next}
  onswiperight={prev}
>...</div>
```

类型参数：
1. 节点类型（默认 `Element`）
2. 参数类型
3. 该 action 派发的自定义事件

## 4. 旧式 update/destroy（兼容）

```svelte
<script>
  /** @type {import('svelte/action').Action} */
  function legacy(node, data) {
    return {
      update(newData) { /* 参数变化时调用 */ },
      destroy() { /* 卸载 */ }
    };
  }
</script>

<div use:legacy={{ x: 1 }}>...</div>
```

> Svelte 5 推荐用 `$effect` 内返回清理函数的形式。

## 5. 实际示例：点击外部关闭

```svelte
<script>
  /** @type {import('svelte/action').Action<HTMLElement, () => void>} */
  function clickOutside(node, callback) {
    function handle(e) {
      if (!node.contains(e.target)) callback();
    }
    $effect(() => {
      document.addEventListener('click', handle, true);
      return () => document.removeEventListener('click', handle, true);
    });
  }

  let open = $state(false);
</script>

<div use:clickOutside={() => open = false}>
  {#if open}
    <div class="dropdown">menu</div>
  {/if}
</div>
```

## 6. 实际示例：自动 focus

```svelte
<script>
  /** @type {import('svelte/action').Action<HTMLInputElement>} */
  function autofocus(node) {
    $effect(() => {
      node.focus();
    });
  }
</script>

<input use:autofocus />
```

## 7. Action vs Attachment 关键差异

| 行为 | `use:` | `{@attach}` |
|------|--------|-------------|
| SSR | 不调用 | 不调用 |
| 参数变化 | 不重新运行 | 重新运行 |
| 用于组件 | 不会传到子元素 | 通过展开 prop 传递 |
| 响应式内部 state | 需 `$effect` 包装 | 自动响应 |

## 8. 迁移 use: → {@attach}

```svelte
<!-- 旧 -->
<div use:tooltip={content}></div>

<!-- 新 -->
<div {@attach tooltip(content)}></div>

<!-- 如果原 action 库不更新，可用 fromAction 包装 -->
<script>
  import { fromAction } from 'svelte/attachments';
  import { tooltipAction } from 'lib';
  const tooltip = fromAction(tooltipAction);
</script>
<div {@attach tooltip(content)}></div>
```
