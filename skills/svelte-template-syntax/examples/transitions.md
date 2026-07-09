# transition: / in: / out:

Transitions 监听元素进入/离开 DOM 时的状态变化。`transition:` 是**双向**（可中途反向），`in:` 和 `out:` 是**单向**。

## 1. 基本 transition

```svelte
<script>
  import { fade } from 'svelte/transition';
  let visible = $state(false);
</script>

<button onclick={() => visible = !visible}>toggle</button>

{#if visible}
  <div transition:fade>fades in and out</div>
{/if}
```

> 块（如 `{#if}`）卸载时，块内所有元素（含无 transition 的）会保留在 DOM 中直到所有 transition 完成。

## 2. 局部 vs 全局

Transitions 默认是**局部**的：仅在所属块创建/销毁时触发，不响应父块变化。

```svelte
{#if x}
  {#if y}
    <p transition:fade>仅 y 变化时触发</p>
    <p transition:fade|global>x 或 y 变化时触发</p>
  {/if}
{/if}
```

## 3. 内置 transition 列表

从 `svelte/transition` 导入：

```svelte
<script>
  import {
    fade,        // 透明度
    blur,        // 模糊 + 透明度
    fly,         // 任意方向位移
    slide,       // 水平/垂直滑动
    scale,       // 缩放
    draw,        // SVG stroke 绘制
    crossfade    // 元素之间交叉淡入淡出
  } from 'svelte/transition';
</script>

{#if visible}
  <div transition:fade>...</div>
  <div transition:fly={{ y: 200, duration: 500 }}>...</div>
  <div transition:blur={{ amount: 10 }}>...</div>
  <div transition:scale={{ start: 0.5 }}>...</div>
  <div transition:slide>...</div>
{/if}
```

## 4. Transition 参数

```svelte
<!-- 双花括号是对象字面量，不是特殊语法 -->
<div transition:fade={{ duration: 2000 }}>slow fade</div>
```

| 字段 | 含义 | 适用 |
|------|------|------|
| `duration` | 毫秒 | 全部 |
| `delay` | 延迟毫秒 | 全部 |
| `easing` | `(t) => number` | 全部 |
| `opacity` | `0..1` 起始透明度 | fade |
| `y`/`x` | 位移量 | fly |
| `start` | `0..1` 起始 scale | scale |
| `amount` | 模糊 px | blur |

## 5. in: / out: 单向

```svelte
<script>
  import { fade, fly } from 'svelte/transition';
  let visible = $state(false);
</script>

{#if visible}
  <div in:fly={{ y: 200 }} out:fade>flies in, fades out</div>
{/if}
```

区别：
- `in:` 继续播放与 `out:` 同时进行（不互相打断）
- `transition:` 可在中途反转

## 6. 自定义 transition 函数

```svelte
<script>
  import { elasticOut } from 'svelte/easing';

  function whoosh(node, params) {
    const existing = getComputedStyle(node).transform.replace('none', '');
    return {
      delay: params.delay || 0,
      duration: params.duration || 400,
      easing: params.easing || elasticOut,
      css: (t, u) => `transform: ${existing} scale(${t})`
    };
  }
</script>

{#if visible}
  <div in:whoosh>whoosh</div>
{/if}
```

> 优先用 `css`（Web Animations 跑在主线程外）而不是 `tick`。

## 7. tick 函数（每帧 JS）

```svelte
<script>
  function typewriter(node, { speed = 1 }) {
    const valid = node.childNodes.length === 1 &&
                  node.childNodes[0].nodeType === Node.TEXT_NODE;
    if (!valid) throw new Error('need single text node');

    const text = node.textContent;
    const duration = text.length / (speed * 0.01);

    return {
      duration,
      tick: (t) => {
        const i = ~~(text.length * t);
        node.textContent = text.slice(0, i);
      }
    };
  }
</script>

{#if visible}
  <p in:typewriter={{ speed: 1 }}>The quick brown fox...</p>
{/if}
```

## 8. options.direction

`options` 包含 `direction: 'in' | 'out' | 'both'`，适合 `transition:` 双向使用：

```js
function myFade(node, params, options) {
  return {
    duration: 400,
    css: (t, u) => `opacity: ${t}` // t: 0→1 入场，1→0 出场
  };
}
```

## 9. Transition 事件

```svelte
<script>
  import { fly } from 'svelte/transition';
  let visible = $state(false);
  let status = $state('');
</script>

{#if visible}
  <p
    transition:fly={{ y: 200, duration: 2000 }}
    onintrostart={() => status = 'intro started'}
    onoutrostart={() => status = 'outro started'}
    onintroend={() => status = 'intro ended'}
    onoutroend={() => status = 'outro ended'}
  >
    Flies
  </p>
{/if}
<p>{status}</p>
```

## 10. 跨元素 crossfade

```svelte
<script>
  import { crossfade } from 'svelte/transition';
  import { quintOut } from 'svelte/easing';

  const [send, receive] = crossfade({
    duration: d => Math.sqrt(d * 200),
    fallback(node, params) {
      const style = getComputedStyle(node);
      const transform = style.transform === 'none' ? '' : style.transform;
      return {
        duration: 600,
        easing: quintOut,
        css: t => `
          transform: ${transform} scale(${t});
          opacity: ${t}
        `
      };
    }
  });
</script>

{#if items}
  {#each items as item (item.id)}
    <li
      in:receive={{ key: item.id }}
      out:send={{ key: item.id }}
    >{item.text}</li>
  {/each}
{/if}
```

## 11. 列表过渡

`{#each}` 块内的元素 transition 在添加/删除/移动时触发：

```svelte
{#each items as item (item.id)}
  <div transition:slide>{item.text}</div>
{/each}
```

## 12. 配合 {#key} 强制重启

```svelte
{#key value}
  <div transition:fade>{value}</div>
{/key}
```

## 13. Transition 不能跨 SSR 边界

Transitions 只在客户端运行，SSR 期间直接渲染最终状态。
