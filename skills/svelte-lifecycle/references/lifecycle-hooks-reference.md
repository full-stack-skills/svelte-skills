# Lifecycle Hooks Reference

Svelte 5 中组件生命周期只有两部分：**创建** 与 **销毁**。中间的状态更新不是组件级事件，而是各个 render effect 的事件。

## 导入

```js
import { onMount, onDestroy, tick } from 'svelte';
```

> `beforeUpdate` / `afterUpdate` 在 Svelte 5 中**已 deprecated**，仅在未使用 runes 的传统组件中可用作 shim。

## onMount

**签名**：`onMount(fn: () => any): void`

- 调用时机：组件挂载到 DOM 后
- **SSR：不执行**
- 返回值：若为函数，则在组件卸载时调用

### 同步函数约束

```svelte
<script>
  import { onMount } from 'svelte';

  // ✅ 同步函数，cleanup 会被调用
  onMount(() => {
    const id = setInterval(() => {}, 1000);
    return () => clearInterval(id);
  });

  // ❌ async 函数，cleanup 不会执行
  onMount(async () => {
    const data = await fetch(...);
    return () => {};  // 不会调用
  });
</script>
```

> TypeScript 中 async onMount 会报类型错误（自 Svelte 4）。

### 从外部模块调用

```js
// analytics.js
import { onMount } from 'svelte';

export function trackPage(name) {
  onMount(() => {
    console.log(`view: ${name}`);
    return () => console.log(`leave: ${name}`);
  });
}
```

`onMount` 调用必须在**组件初始化**期间（同步、声明式），但不必写在组件脚本顶层。

## onDestroy

**签名**：`onDestroy(fn: () => any): void`

- 调用时机：组件卸载前
- **SSR：执行**（onMount/beforeUpdate/afterUpdate 都不在 SSR 执行）

```svelte
<script>
  import { onDestroy } from 'svelte';
  onDestroy(() => console.log('cleanup'));
</script>
```

## tick

**签名**：`tick(): Promise<void>`

- 返回 `Promise`，在所有 pending state 变更应用到 DOM 后 resolve
- 若无 pending state 变更，则在下一个 microtask resolve

```svelte
<script>
  import { tick } from 'svelte';

  async function scroll() {
    items.push(newItem);
    await tick();
    viewport.scrollTop = viewport.scrollHeight;
  }
</script>
```

常用于：focus、scroll、measure DOM、调用命令式 API。

## beforeUpdate / afterUpdate（deprecated）

```svelte
<script>
  import { beforeUpdate, afterUpdate } from 'svelte';

  beforeUpdate(() => console.log('about to update'));
  afterUpdate(() => console.log('just updated'));
</script>
```

**Svelte 5 替代**：

| Svelte 4 | Svelte 5 |
|---|---|
| `beforeUpdate(() => { ... })` | `$effect.pre(() => { ... })` |
| `afterUpdate(() => { ... })` | `$effect(() => { ... })` |

Runes 版只对**显式读取**的状态变化触发，避免无关更新。

## 清理顺序

组件卸载时：
1. `onMount` 返回的 cleanup 函数
2. `onDestroy` 回调
3. `$effect` 的 cleanup 函数

## SSR 总结

| 钩子 | 浏览器 | SSR |
|---|---|---|
| `onMount` | ✅ | ❌ |
| `onDestroy` | ✅ | ✅ |
| `tick` | ✅ | ✅ |
| `beforeUpdate` (legacy) | ✅ | ❌ |
| `afterUpdate` (legacy) | ✅ | ❌ |

## 实际使用模式

### 测量 DOM

```svelte
<script>
  import { onMount, tick } from 'svelte';

  let box;
  let width = $state(0);

  onMount(() => {
    width = box.offsetWidth;
  });
</script>
```

### 集成第三方库

```svelte
<script>
  import { onMount } from 'svelte';
  import tippy from 'tippy.js';

  let anchor;
  onMount(() => {
    const inst = tippy(anchor);
    return () => inst.destroy();
  });
</script>

<button bind:this={anchor}>Hover</button>
```

### AbortController 取消 fetch

```svelte
<script>
  import { onMount } from 'svelte';

  let data = $state(null);

  onMount(() => {
    const ac = new AbortController();
    fetch('/api', { signal: ac.signal })
      .then(r => r.json())
      .then(d => data = d);
    return () => ac.abort();
  });
</script>
```
