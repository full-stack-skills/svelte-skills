# animate:

`animate:` 指令在 keyed `{#each}` 块的**已有元素**重排时触发动画。新增/删除不会触发动画 — `animate:` 必须在 keyed each 的**直接子元素**上。

## 1. 基本 animate:flip

```svelte
<script>
  import { flip } from 'svelte/animate';
  let list = $state([1, 2, 3, 4, 5]);
</script>

<button onclick={() => list = list.toSorted(() => Math.random() - 0.5)}>
  shuffle
</button>

{#each list as item (item)}
  <li animate:flip>{item}</li>
{/each}
```

## 2. Animation 参数

```svelte
{#each list as item (item)}
  <li animate:flip={{ delay: 500, duration: 300, easing: cubicOut }}>
    {item}
  </li>
{/each}
```

| 字段 | 含义 |
|------|------|
| `delay` | 延迟毫秒 |
| `duration` | 毫秒 |
| `easing` | `(t) => number` |
| `css` 或 `tick` | 渲染方式 |

## 3. 自定义 animation 函数

```svelte
<script>
  import { cubicOut } from 'svelte/easing';

  function whizz(node, { from, to }, params) {
    const dx = from.left - to.left;
    const dy = from.top - to.top;
    const d = Math.sqrt(dx * dx + dy * dy);

    return {
      delay: 0,
      duration: Math.sqrt(d) * 120,
      easing: cubicOut,
      css: (t, u) => `transform: translate(${u * dx}px, ${u * dy}px) rotate(${t * 360}deg);`
    };
  }
</script>

{#each list as item, index (item)}
  <div animate:whizz>{item}</div>
{/each}
```

`from` / `to` 是 `DOMRect`，描述元素的起始和结束位置。

## 4. 用 tick 替代 css

```svelte
<script>
  import { cubicOut } from 'svelte/easing';

  function colorFlip(node, { from, to }, params) {
    const dx = from.left - to.left;
    const dy = from.top - to.top;
    const d = Math.sqrt(dx * dx + dy * dy);

    return {
      delay: 0,
      duration: Math.sqrt(d) * 120,
      easing: cubicOut,
      tick: (t, u) => {
        node.style.color = t > 0.5 ? 'Pink' : 'Blue';
      }
    };
  }
</script>

{#each list as item (item)}
  <div animate:colorFlip>{item}</div>
{/each}
```

> 优先用 `css`（Web Animations 跑在主线程外）而不是 `tick`。

## 5. animate:flip + 列表过渡组合

```svelte
<script>
  import { flip } from 'svelte/animate';
  import { send, receive } from './crossfade.js';
  import { fade } from 'svelte/transition';

  let items = $state(['a', 'b', 'c', 'd']);
</script>

{#each items as item (item)}
  <li
    animate:flip={{ duration: 300 }}
    in:receive={{ key: item }}
    out:send={{ key: item }}
  >{item}</li>
{/each}
```

## 6. 排序后保留位置

```svelte
<script>
  import { flip } from 'svelte/animate';
  let items = $state([
    { id: 1, label: 'First' },
    { id: 2, label: 'Second' },
    { id: 3, label: 'Third' }
  ]);

  function reverse() {
    items = items.toReversed();
  }
</script>

<button onclick={reverse}>reverse</button>

<ul>
  {#each items as item (item.id)}
    <li animate:flip={{ duration: 400 }}>{item.label}</li>
  {/each}
</ul>
```

## 7. 与 {#key} 区分

| 场景 | 用什么 |
|------|--------|
| 元素进入/离开 DOM | `transition:` / `in:` / `out:` |
| 已有元素重排 | `animate:` |
| 表达式变化整体重建 | `{#key expr}` |

## 8. 动画与 keyed each 绑定

`animate:` 必须紧邻 keyed each 子元素：

```svelte
<!-- ❌ 错误：不在 keyed each 内 -->
<div animate:flip>...</div>

<!-- ✅ 正确 -->
{#each list as item (item.id)}
  <div animate:flip>...</div>
{/each}
```

## 9. 内置 animation 列表

从 `svelte/animate` 导入：

- `flip` — First-Last-In-First-Out 平滑移动
- 可自定义任意函数

```svelte
<script>
  import { flip } from 'svelte/animate';
</script>
```

## 10. 性能提示

- `flip` 默认使用 `transform` — 触发 GPU 合成
- 避免在大型列表上同时触发过多动画
- 长列表考虑虚拟化（virtualization）
