# animate: Reference

## 触发条件

`animate:` 在 **keyed `{#each}` 块的已有元素重排**时触发动画。

- **不**在元素新增/删除时触发（用 `transition:` / `in:` / `out:`）
- **必须**是 keyed each 的**直接子元素**

```svelte
{#each list as item, index (item)}
  <li animate:flip>{item}</li>
{/each}
```

## 内置 animations（`svelte/animate`）

| 名称 | 描述 |
|------|------|
| `flip` | First-Last-In-First-Out 平滑移动（默认） |

## Animation 参数

```svelte
<li animate:flip={{ delay: 500, duration: 300, easing: cubicOut }}>...</li>
```

通用字段：
- `delay: number`
- `duration: number`
- `easing: (t: number) => number`

## 自定义 animation 函数

```ts
type AnimationFn = (
  node: HTMLElement,
  { from, to }: { from: DOMRect; to: DOMRect },
  params: any
) => AnimationConfig | (() => AnimationConfig);

interface AnimationConfig {
  delay?: number;
  duration?: number;
  easing?: (t: number) => number;
  css?: (t: number, u: number) => string;
  tick?: (t: number, u: number) => void;
}
```

### from / to

- `from` — 元素在**起始位置**的 `DOMRect`
- `to` — 元素在**重排后最终位置**的 `DOMRect`

测量的是渲染前后的几何位置。

### css vs tick

- **`css(t, u)`** → 字符串；生成 Web Animations（离主线程）
- **`tick(t, u)`** → 每帧 JS 回调

> 优先用 `css`。

## 内置 vs 自定义对照

| 场景 | 推荐 |
|------|------|
| 简单列表重排 | `animate:flip` |
| 旋转 + 移动 | 自定义 `css: translate + rotate` |
| 颜色变化 | 自定义 `tick: color = t > 0.5 ? 'a' : 'b'` |
| 复杂路径 | 自定义函数 |

## 与 transitions 配合

列表综合使用：重排走 `animate:`，新增/删除走 `transition:`。

```svelte
<script>
  import { flip } from 'svelte/animate';
  import { fade } from 'svelte/transition';
  import { send, receive } from './crossfade.js';

  let items = $state(['a', 'b', 'c']);
</script>

{#each items as item (item)}
  <li
    animate:flip={{ duration: 300 }}
    in:receive={{ key: item }}
    out:send={{ key: item }}
  >{item}</li>
{/each}
```

## 关键规则

1. `animate:` 必须在 keyed each 内（`{#each list as item (key)}`）
2. 必须是 each 块的**直接子元素**
3. 不读响应式 state 不会触发重跑
4. SSR 期间不运行
5. 元素无 DOM 尺寸（`display: inline`）时无法测量

## 性能建议

- `flip` 默认使用 `transform` → GPU 合成
- 长列表（>100）考虑虚拟化以减少同时动画数量
- 简单的 `easing` 函数更高效
- 避免在动画中修改 `box` / `width` / `height`（会触发布局）

## 错误对照

| 错误 | 原因 |
|------|------|
| 动画不触发 | each 块未 keyed |
| 元素抖一下 | transition 和 animate 冲突 |
| `flip` 报错 | 元素无尺寸 |
| 新增不动画 | 用错 `transition:` 而非 `animate:` |
| 类型错误 | 自定义函数签名错 |

## 与 #each 关键

```svelte
<!-- ❌ 无 key：不触发 animate -->
{#each list as item}
  <li animate:flip>{item}</li>
{/each}

<!-- ✅ 有 key -->
{#each list as item (item.id)}
  <li animate:flip>{item}</li>
{/each}
```

key 必须是稳定唯一值（string/number 优先）。
