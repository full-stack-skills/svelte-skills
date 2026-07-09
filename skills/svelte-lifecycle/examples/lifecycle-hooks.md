# Lifecycle Hooks Examples

## 1. onMount 基础

```svelte
<script>
  import { onMount } from 'svelte';

  onMount(() => {
    console.log('component mounted');
  });
</script>
```

## 2. onMount 启动定时器 + cleanup

```svelte
<script>
  import { onMount } from 'svelte';
  import { writable } from 'svelte/store';

  const time = writable(0);

  onMount(() => {
    const id = setInterval(() => time.update(t => t + 1), 1000);
    return () => clearInterval(id);  // 组件卸载时清理
  });
</script>

<p>Seconds: {$time}</p>
```

## 3. onMount 异步获取数据（正确清理）

```svelte
<script>
  import { onMount } from 'svelte';

  let data = $state(null);
  let error = $state(null);

  onMount(() => {
    let cancelled = false;
    (async () => {
      try {
        const r = await fetch('/api/me');
        if (!cancelled) data = await r.json();
      } catch (e) {
        if (!cancelled) error = e.message;
      }
    })();
    return () => { cancelled = true; };
  });
</script>

{#if data}<p>{data.name}</p>
{:else if error}<p>Error: {error}</p>
{:else}<p>Loading…</p>{/if}
```

> 注意：`onMount` 函数本身必须**同步**——`async` 会导致 cleanup 失效。

## 4. onMount 绑定 window 事件

```svelte
<script>
  import { onMount } from 'svelte';

  let scrollY = $state(0);

  onMount(() => {
    const handler = () => scrollY = window.scrollY;
    window.addEventListener('scroll', handler, { passive: true });
    return () => window.removeEventListener('scroll', handler);
  });
</script>

<p>Scroll: {scrollY}px</p>
```

## 5. onMount 设置第三方库

```svelte
<script>
  import { onMount } from 'svelte';
  import tippy from 'tippy.js';

  let anchor;

  onMount(() => {
    const instance = tippy(anchor, { content: 'Hello!' });
    return () => instance.destroy();
  });
</script>

<button bind:this={anchor}>Hover me</button>
```

## 6. onDestroy 在 SSR 中也会执行

```svelte
<script>
  import { onDestroy } from 'svelte';

  onDestroy(() => {
    // 浏览器和 SSR 都会执行
    console.log('cleanup');
  });
</script>
```

常见用途：关闭数据库连接、释放文件句柄、记录日志。

## 7. tick 等待 DOM 更新后 focus

```svelte
<script>
  import { tick } from 'svelte';

  let show = $state(false);
  let input;

  async function reveal() {
    show = true;
    await tick();        // 等 #if 块渲染
    input?.focus();
  }
</script>

<button onclick={reveal}>Show input</button>
{#if show}
  <input bind:this={input} placeholder="Focus me" />
{/if}
```

## 8. tick 在 DOM 更新后测量

```svelte
<script>
  import { tick } from 'svelte';

  let text = $state('hello');
  let box;
  let width = $state(0);

  $effect(() => {
    text;                  // 显式追踪
    tick().then(() => {
      width = box?.offsetWidth ?? 0;
    });
  });
</script>

<button onclick={() => text += ' world'}>Grow</button>
<div bind:this={box}>{text}</div>
<p>Width: {width}px</p>
```

## 9. onMount 集成 WebSocket

```svelte
<script>
  import { onMount } from 'svelte';
  import { writable } from 'svelte/store';

  export let url;
  const messages = writable([]);

  onMount(() => {
    const ws = new WebSocket(url);

    ws.addEventListener('message', (e) => {
      messages.update(arr => [...arr, e.data]);
    });

    return () => ws.close();
  });
</script>

<ul>
  {#each $messages as m}
    <li>{m}</li>
  {/each}
</ul>
```

## 10. 迁移：beforeUpdate → $effect.pre

```svelte
<script>
  import { tick } from 'svelte';

  let theme = $state('dark');
  let messages = $state([]);
  let viewport;

  // Svelte 4 风格：每次组件更新都跑
  // beforeUpdate(() => { ... });

  // Svelte 5 风格：只在 messages 变化时跑
  $effect.pre(() => {
    messages;  // 显式依赖
    const autoscroll = viewport &&
      viewport.offsetHeight + viewport.scrollTop >
      viewport.scrollHeight - 50;
    if (autoscroll) {
      tick().then(() => viewport.scrollTo(0, viewport.scrollHeight));
    }
  });
</script>

<div bind:this={viewport}>
  {#each messages as m}<p>{m}</p>{/each}
</div>
```

## 11. onMount 调用链：从外部模块

```js
// analytics.js
import { onMount } from 'svelte';

export function trackPageView(name) {
  onMount(() => {
    console.log(`view: ${name}`);
    return () => console.log(`leave: ${name}`);
  });
}
```

```svelte
<!-- Page.svelte -->
<script>
  import { trackPageView } from './analytics.js';
  trackPageView('home');
</script>
```

> 关键：`onMount` 调用必须在**组件初始化**期间，但不必写在组件脚本顶层。

## 12. 错误：async onMount 失效

```svelte
<script>
  import { onMount } from 'svelte';

  // ❌ cleanup 不会执行
  // onMount(async () => {
  //   const id = setInterval(() => console.log('tick'), 1000);
  //   return () => clearInterval(id);  // 永远不调用
  // });

  // ✅ 正确写法
  onMount(() => {
    const id = setInterval(() => console.log('tick'), 1000);
    return () => clearInterval(id);
  });
</script>
```
