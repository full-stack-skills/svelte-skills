---
name: svelte-runtime
description: Svelte 5 运行时技能。当用户需要使用 Stores、Context、生命周期钩子（onMount/onDestroy/tick）、命令式组件 API（mount/unmount/render/hydrate）、Hydratable Data、Best Practices 或 Vitest 测试时使用。
---

# Svelte Runtime Reference (Svelte 5)

本技能覆盖 Svelte 5 的运行时 API，包括跨组件状态（Stores/Context）、生命周期、命令式 API、SSR 和测试。

## When to use this skill

当用户需要跨组件共享状态、处理组件生命周期、使用 `mount`/`render` 等命令式 API、或编写 Vitest 组件测试时使用本技能。

## Critical: Stores (svelte/store)

Store 是一种跨组件共享响应式状态的老方式（Svelte 5 推荐用 `.svelte.js` 中的 `$state` 对象替代）。

### writable

```js
import { writable } from 'svelte/store';
const count = writable(0);

count.subscribe(v => console.log(v)); // 订阅
count.set(1);                         // 设置值
count.update(n => n + 1);            // 更新
```

### readable（不可从外部设置）

```js
import { readable } from 'svelte/store';
const time = readable(new Date(), (set) => {
  const interval = setInterval(() => set(new Date()), 1000);
  return () => clearInterval(interval);
});
```

### derived

```js
import { derived } from 'svelte/store';
const doubled = derived(count, $c => $c * 2);
const combined = derived([a, b], ([$a, $b]) => $a + $b);
```

### readonly

```js
import { readonly, writable } from 'svelte/store';
const readOnlyStore = readonly(writableStore);
```

### 在组件中使用

```svelte
<script>
  import { count } from './store.js';
  console.log($count); // $ 前缀订阅
  $count = 2;          // 自动调用 .set()
</script>
```

> Store 必须在组件顶层声明；不要在变量前加 `$` 前缀（那是 `$count`，不是 `count`）。

**Svelte 5 推荐替代方案**：
```js
// state.svelte.js
export const userState = $state({ name: '', count: 0 });
// 其他组件直接 import
```

## Critical: Context (svelte)

跨组件层级传递数据，无需 props 层层透传（prop-drilling）：

### setContext / getContext

```svelte
// Parent.svelte
<script>
  import { setContext } from 'svelte';
  setContext('key', 'hello');
</script>

// Child.svelte
<script>
  import { getContext } from 'svelte';
  const msg = getContext('key');
</script>
```

### createContext（推荐，类型安全）

```ts
// context.ts
import { createContext } from 'svelte';
export const [getUserContext, setUserContext] = createContext<User>();
```

```svelte
// Parent.svelte
<script> setUserContext({ name: 'Ada' }); </script>

// Child.svelte
<script> const user = getUserContext(); </script>
```

### 响应式 Context

将 `$state` 对象存入 Context：

```svelte
setContext('counter', counterState);
// ❌ 不要重新赋值整个对象
// counterState = { count: 0 }
// ✅ 直接修改属性
counterState.count = 0;
```

## Critical: Lifecycle Hooks

### onMount

组件挂载到 DOM 后执行（SSR 时不运行）：

```svelte
<script>
  import { onMount } from 'svelte';
  onMount(() => {
    console.log('mounted');
    return () => console.log('unmounted'); // 返回清理函数
  });
</script>
```

> 返回的函数在组件卸载时调用；如果传入 async 函数则清理函数不会被调用。

### onDestroy

组件卸载前执行（SSR 时也运行）：

```svelte
<script>
  import { onDestroy } from 'svelte';
  onDestroy(() => cleanup());
</script>
```

### tick

等待 DOM 更新完成后再继续：

```svelte
<script>
  import { tick } from 'svelte';
  $effect.pre(() => {
    // DOM 更新前
    tick().then(() => {
      // DOM 已更新完成
    });
  });
</script>
```

## Critical: Imperative Component API

### mount（客户端挂载）

```js
import { mount } from 'svelte';
import App from './App.svelte';

const app = mount(App, {
  target: document.querySelector('#app'),
  props: { name: 'world' }
});
```

### unmount（卸载）

```js
import { mount, unmount } from 'svelte';
unmount(app, { outro: true }); // 播放过渡后卸载
```

### render（服务端渲染）

```js
import { render } from 'svelte/server';
const { head, body } = await render(App, { props: { data } });
```

### hydrate（客户端水合）

复用 SSR HTML 并激活为交互式：

```js
import { hydrate } from 'svelte';
const app = hydrate(App, { target: document.querySelector('#app') });
```

## Critical: Hydratable Data

SSR 时序列化数据，客户端水合时复用（避免重复请求）：

```svelte
<script>
  import { hydratable } from 'svelte';
  const user = await hydratable('user', () => getUser());
</script>
<h1>{user.name}</h1>
```

### CSP 支持

```js
const { head, body } = await render(App, {
  csp: { nonce: 'xxx' } // 或 hash: true
});
```

## Critical: Best Practices

### $state

```js
// ✅ 只对需要响应式的变量用 $state
let count = $state(0);

// ✅ 大数组/不可变数据用 $state.raw
let apiData = $state.raw([]);
apiData = await fetchData(); // 重新赋值触发更新
```

### $derived

```js
// ✅ 派生值用 $derived
let doubled = $derived(count * 2);

// ❌ 不要用 $effect 同步派生
let square;
$effect(() => { square = num * num; }); // ❌
let square = $derived(num * num);         // ✅
```

### $effect

```js
// ✅ 仅用于副作用
$effect(() => { myChart.resize(); });

// ❌ 不要同步状态
$effect(() => { left = total - spent; }); // ❌
```

### Props

```js
// ✅ 依赖 props 的值用 $derived
let { type } = $props();
let color = $derived(type === 'danger' ? 'red' : 'green');
```

### 事件

```svelte
<!-- ✅ 现代事件属性 -->
<button onclick={handleClick}>click</button>

<!-- ❌ 旧式 on:click -->
<button on:click={handleClick}>click</button>

<!-- ✅ 监听 window 事件 -->
<svelte:window onkeydown={handleKey} />
```

### Snippets

```svelte
<!-- ✅ 推荐：使用 snippet -->
{#snippet footer()}
  <p>footer</p>
{/snippet}
```

### Each blocks

```svelte
<!-- ✅ 始终使用 key -->
{#each items as item (item.id)}
  <li>{item.name}</li>
{/each}
```

### Context

```js
// ✅ 用 createContext 提供类型安全
import { createContext } from 'svelte';
const [getCtx, setCtx] = createContext<MyCtx>();
```

### Avoid Legacy Features

```js
// ✅ Runes mode 全是新项目的默认
let count = $state(0);     // ✅
$: count++;                  // ❌ legacy
export let name;            // ❌
let { name } = $props();   // ✅
on:click={fn}              // ❌
onclick={fn}               // ✅
<slot />                   // ❌
{@render children()}        // ✅
<svelte:component this={x} /> // ❌
<svelte:element this={x} />   // ✅
```

## Critical: Testing (Vitest)

### 安装配置

```bash
npm install -D vitest
```

```js
// vite.config.js
export default defineConfig({
  resolve: process.env.VITEST
    ? { conditions: ['browser'] }
    : undefined
});
```

### 单元测试

```js
// multiplier.svelte.js
export function multiplier(initial, k) {
  let count = $state(initial);
  return {
    get value() { return count * k; },
    set: (c) => { count = c; }
  };
}

// test
import { multiplier } from './multiplier.svelte.js';
test('multiplier', () => {
  let double = multiplier(0, 2);
  expect(double.value).toEqual(0);
  double.set(5);
  expect(double.value).toEqual(10);
});
```

### 组件测试

```js
import { mount, unmount } from 'svelte';
import { expect, test } from 'vitest';

test('MyComponent', () => {
  const comp = mount(MyComponent, { target: document.body, props: { name: 'Ada' } });
  expect(document.body.innerHTML).toContain('Ada');
  unmount(comp);
});
```

### flushSync

强制同步执行所有待处理效果（测试中常用）：

```js
import { flushSync } from 'svelte';
flushSync();
```

## Quick Fixes

| 问题 | 解决方案 |
|------|----------|
| 状态在组件间不同步 | 用 Context（跨层级）或 `.svelte.js`（全局） |
| onMount 清理不执行 | 确保传入同步函数（不是 async） |
| SSR 后组件不交互 | 用 `hydrate` 而非 `mount` |
| 测试中状态不更新 | 用 `flushSync()` 强制同步执行 |
| Context 断开链接 | 不要重新赋值整个 Context 对象 |

## Gotchas

1. **Store `$` 前缀是订阅语法** — `$store` 在模板和 script 中都是订阅，非 `$` 前缀的变量是普通值
2. **Context 链接断开** — 重新赋值整个对象会断开响应式链接，改用 `.count = 0` 而非 `obj = {count:0}`
3. **`mount` 不运行 `$effect`** — effects 在组件挂载后由 microtask 触发
4. **测试中 `$effect.root`** — 涉及 effects 的测试需要 `$effect.root()` 包裹
5. **`render` 只在 SSR 环境可用** — 客户端用 `mount`

## FAQ

**Q: 什么时候用 Store vs Context vs .svelte.js？**
A: Store：跨全应用共享的简单值；Context：组件树局部共享；`.svelte.js` `$state`：推荐的新方式，可替代 Store。

**Q: `tick` 和 `await Promise.resolve()` 有什么区别？**
A: `tick()` 保证在 DOM 更新后解析，`Promise.resolve()` 不等待 Svelte 的更新周期。

**Q: 如何测试带有 `$effect` 的代码？**
A: 用 `flushSync()` 强制执行，使用 `$effect.root()` 包裹测试代码。

**Q: SSR 后 hydration 报错？**
A: 确保 SSR 和 CSR 的 HTML 结构一致；用 `hydrate` 而非 `mount`。

## Examples

Practical examples demonstrating Svelte 5's runtime APIs.

| File | Description |
|------|-------------|
| [examples/README.md](./examples/README.md) | Entry point with quick reference |
| [examples/mount-hydrate.md](./examples/mount-hydrate.md) | 9 examples: mount, unmount, hydrate, render, flushSync, tick |
| [examples/ssr-patterns.md](./examples/ssr-patterns.md) | SSR patterns: async render, streaming, hydration, CSP |

## References

In-depth technical references for Svelte 5's runtime APIs.

| File | Description |
|------|-------------|
| [references/README.md](./references/README.md) | Entry point with quick reference |
| [references/mount-unmount-reference.md](./references/mount-unmount-reference.md) | Deep dive: mount/unmount/hydrate lifecycle, flushSync, tick |
| [references/ssr-reference.md](./references/ssr-reference.md) | SSR deep dive: render, async SSR, CSP, hydration |
