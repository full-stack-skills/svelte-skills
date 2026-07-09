---
name: svelte-lifecycle
description: Svelte 5 生命周期、Stores、Context、Testing 技能。当用户需要在 Svelte 5 中使用 onMount/onDestroy/tick 生命周期钩子、使用 writable/derived stores 管理状态、使用 createContext 共享组件树状态、编写 Vitest/Storybook/Playwright 测试时使用。
---

# Svelte Lifecycle, Stores, Context, Testing (Svelte 5)

本技能覆盖 Svelte 5 中"非 Runes" 但仍属于核心响应式架构的子系统：生命周期钩子（`onMount` / `onDestroy` / `tick`）、Stores（`writable` / `readable` / `derived` / 自定义）、Context API（`createContext` / `setContext` / `getContext`），以及测试体系（Vitest 单元与组件测试、Storybook、Playwright e2e）。也覆盖浏览器支持矩阵与例外。

## When to use this skill

- 需要在组件挂载后访问 DOM（`onMount`）或在组件销毁前清理（`onDestroy`）
- 需要在状态变更后等待 DOM 更新（`tick`）
- 需要跨组件共享反应式状态，但不想用 `import` 模块（用 Stores 或 Context）
- 需要写跨组件异步流（Readable/derived store）
- 需要用 Context 替代 prop drilling，且涉及 SSR 时不污染全局
- 需要为 Svelte 组件写 Vitest 单元/组件测试、Storybook stories 或 Playwright e2e
- 需要查浏览器最低版本与例外特性

## Critical: onMount

`onMount(fn)` 在组件挂载到 DOM 后立即运行。**仅在浏览器执行**——SSR 时不调用。

### 基础用法

```svelte
<script>
  import { onMount } from 'svelte';

  onMount(() => {
    console.log('mounted');
  });
</script>
```

### 返回清理函数

```svelte
<script>
  import { onMount } from 'svelte';

  onMount(() => {
    const interval = setInterval(() => console.log('beep'), 1000);
    return () => clearInterval(interval);
  });
</script>
```

> **关键约束**：`onMount` 必须接收**同步函数**才能正确返回 cleanup。`async () => {}` 总是返回 `Promise`，cleanup 不会在卸载时调用。

### 从外部模块调用

`onMount` 不必写在组件脚本顶层——它必须只在组件初始化时调用。允许从同一模块内的 helper 函数中调用。

## Critical: onDestroy

`onDestroy(fn)` 在组件销毁前**立即**执行。**在 SSR 组件中也会运行**——这是四个生命周期钩子中唯一在服务器端执行的。

```svelte
<script>
  import { onDestroy } from 'svelte';

  onDestroy(() => {
    console.log('destroyed');
  });
</script>
```

常见用途：清理 `setInterval`、取消 `fetch` AbortController、解绑全局事件监听。

## Critical: tick

`tick()` 返回 `Promise`，在所有 pending state 变更应用到 DOM 后 resolve；若无 pending 则在下一个 microtask resolve。

```svelte
<script>
  import { tick } from 'svelte';

  async function handle() {
    count = count + 1;
    await tick();
    // 此时 DOM 已更新
    element.scrollIntoView();
  }
</script>
```

常见场景：focus、scroll、measure DOM、在 state 变更后调用第三方命令式 API。

## Deprecated: beforeUpdate / afterUpdate

Svelte 4 时代的"整个组件更新前后"钩子。Svelte 5 中**被 shim 但在 runes 组件中不可用**，应改用：

| Svelte 4 | Svelte 5 |
|---|---|
| `beforeUpdate(() => {})` | `$effect.pre(() => {})` |
| `afterUpdate(() => {})` | `$effect(() => {})` |

Runes 版只在显式读取的状态变化时触发，更精确。例如聊天窗口只在 `messages` 变化时滚到底，主题切换不会重置滚动位置。

## Critical: Stores (writable, readable, derived)

Stores 是满足"store 契约"的对象：必须有 `subscribe(fn) → unsubscribe`；可写 store 还需有 `set(value)`。

```svelte
<script>
  import { writable } from 'svelte/store';

  const count = writable(0);
  $count;        // 自动订阅，读取当前值
  count.set(1);  // 写入
  $count = 2;    // 语法糖：等价于 count.set(2)
</script>
```

### writable

```js
import { writable } from 'svelte/store';

const count = writable(0, () => {
  // 第一个订阅者订阅时调用
  return () => {
    // 最后一个订阅者退订时调用
  };
});
```

第二个参数是 start/stop 函数，常用于建立外部连接（WebSocket、计时器）。

### readable

不可从外部 set 的 store；初始值 + start 函数：

```js
import { readable } from 'svelte/store';

const time = readable(new Date(), (set) => {
  set(new Date());
  const id = setInterval(() => set(new Date()), 1000);
  return () => clearInterval(id);
});
```

### derived

从一个或多个 store 派生：

```js
import { derived, writable } from 'svelte/store';

const a = writable(1);
const b = writable(2);
const sum = derived([a, b], ([$a, $b]) => $a + $b);
```

异步版本：接受 `(values, set, update)`，允许在异步回调里调用 `set`/`update`；可返回清理函数；可传第三个参数作为初始值。

### readonly / get

- `readonly(store)`：包装为只读视图（无 `set`/`update`）
- `get(store)`：同步读一次（内部建立订阅 → 读 → 退订，**不建议在热路径用**）

## When to use stores vs $state

Svelte 5 推荐优先使用 runes（`$state`、`.svelte.js` 模块）：

| 场景 | 推荐 |
|---|---|
| 提取可复用逻辑 | `.svelte.js` 文件 + `$state` |
| 跨组件共享状态 | 模块级 `$state` 对象 |
| 复杂异步数据流 | **Stores**（`readable` + 计时器/订阅） |
| 与 RxJS 互操作 | **Stores**（$ 自动订阅） |
| 跨组件简单计数器 | 两者皆可 |

简单规则：能用 runes 就用 runes，stores 用于"事件流/可观察序列"。

## Critical: Context API

Context 让父组件向任意深度的后代组件传值，无需 prop drilling。

### createContext（推荐，Svelte 5.40+）

```ts
// context.ts
import { createContext } from 'svelte';
interface User { name: string; }
export const [getUser, setUser] = createContext<User>();
```

```svelte
<!-- Parent.svelte -->
<script>
  import { setUser } from './context';
  setUser({ name: 'world' });
</script>
```

```svelte
<!-- Child.svelte -->
<script>
  import { getUser } from './context';
  const user = getUser();
</script>

<h1>hello {user.name}</h1>
```

### setContext / getContext（备选）

```svelte
<!-- Parent -->
<script>
  import { setContext } from 'svelte';
  setContext('my-key', value);
</script>

<!-- Child -->
<script>
  import { getContext } from 'svelte';
  const value = getContext('my-key');
</script>
```

键和值可以是任意 JS 值。`createContext` 提供类型安全与无需 key。

### hasContext / getAllContexts

判断某个 key 是否存在于当前组件上下文层级；`getAllContexts()` 返回所有当前上下文的 Map。常用于库的内部实现。

### 与 state 组合

将 `$state` 对象 set 到 context，**不要重新赋值**——否则破坏响应式链接：

```svelte
<!-- 错误 -->
<button onclick={() => counter = { count: 0 } }>reset</button>

<!-- 正确：原地修改 -->
<button onclick={() => counter.count = 0}>reset</button>
```

Svelte 会发出警告。

### 替代全局 state（SSR 关键）

模块级 `$state` 在 SSR 下会在请求间共享（数据泄漏）。Context 是请求隔离的，因此**涉及用户特定数据时优先用 Context**。

## Critical: Testing

### Vitest 单元测试

```js
// counter.svelte.test.js
import { flushSync } from 'svelte';
import { expect, test } from 'vitest';

test('Counter', () => {
  let count = $state(0);
  count = 1;
  flushSync();
  expect(count).toBe(1);
});
```

### 组件测试

`vite.config.js`：
```js
import { defineConfig } from 'vitest/config';
export default defineConfig({
  test: { environment: 'jsdom' },
  resolve: process.env.VITEST ? { conditions: ['browser'] } : undefined
});
```

```js
import { mount, unmount, flushSync } from 'svelte';
import { expect, test } from 'vitest';
import Component from './Component.svelte';

test('Component', () => {
  const c = mount(Component, { target: document.body, props: { n: 0 } });
  expect(document.body.innerHTML).toBe('<button>0</button>');
  document.body.querySelector('button').click();
  flushSync();
  expect(document.body.innerHTML).toBe('<button>1</button>');
  unmount(c);
});
```

`$effect` 在 `mount` 时不会自动运行——测试中用 `flushSync()` 强制同步触发。

### 使用 $effect 的测试

```js
import { flushSync } from 'svelte';
import { expect, test } from 'vitest';

test('Effect', () => {
  const cleanup = $effect.root(() => {
    let count = $state(0);
    let log = [];
    $effect(() => log.push(count));
    flushSync();
    expect(log).toEqual([0]);
    count = 1;
    flushSync();
    expect(log).toEqual([0, 1]);
  });
  cleanup();
});
```

### Storybook

通过 `npx sv add storybook` 配置；用 `play` 函数模拟用户交互并断言：

```svelte
<Story name="Filled" play={async ({ canvas, userEvent }) => {
  await userEvent.type(canvas.getByTestId('email'), 'a@b.com');
  await userEvent.click(canvas.getByRole('button'));
  await expect(canvas.getByText("You're in!")).toBeInTheDocument();
}} />
```

### Playwright e2e

```js
// tests/home.spec.js
import { expect, test } from '@playwright/test';
test('h1 visible', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('h1')).toBeVisible();
});
```

`playwright.config.js` 中配置 `webServer` 启动预览服务器。

## Critical: Browser support

基线目标 **2020 (Baseline 2020)**：

| 浏览器 | 最低版本 |
|---|---|
| Chrome/Edge | 87 |
| Firefox | 83 |
| Safari | 14 |
| Opera | 73 |
| Opera (Android) | 62 |
| Samsung Internet | 14.0 |
| Android WebView | 87 |
| Internet Explorer | 不支持 |

### 例外特性（需更高版本）

| 特性 | Chrome/Edge | Firefox | Safari |
|---|---|---|---|
| `$state.snapshot` | 98 | 94 | 15.4 |
| `bind:devicePixelContentBoxSize` | — | 93 | 不支持 |
| `flip` from `svelte/animate` | — | 126 | — |

## Quick Fixes

- **onMount 清理不执行** → 函数是 `async`；改用同步函数并内部 `await`。
- **store 引用 UI 不更新** → 必须用 `$store` 前缀读取；不要解构（`const { subscribe } = store` 不会自动订阅）。
- **derived 不触发** → 回调里未读取任何 store 值；或数组写法 `derived([a,b], ([$a,$b]) => ...)`。
- **Context 在 SSR 间泄漏** → 模块级 `$state` 全局共享；改用 `setContext`。
- **Vitest 中 `document is not defined`** → `test.environment: 'jsdom'`；或在文件顶部加 `// @vitest-environment jsdom`。
- **组件 mount 后状态未生效** → 在 mount 后调用 `flushSync()`。
- **Playwright 找不到元素** → 检查 `webServer.command` 是否正确启动；提高 `timeout`。

## Gotchas

- **store 必须在组件顶层声明**：不能在 `if` 块或函数内。
- **不要给非 store 变量加 `$` 前缀**——会被识别为 store。
- **onMount 是唯一的"挂载后"，没有"渲染后"**；用 `tick()` 等待 DOM。
- **onMount 在 SSR 不运行**——访问 `window` / `document` 必须放 onMount 或 onDestroy（onDestroy 在 SSR 运行）。
- **setContext/getContext 必须同步调用**，在 setup 阶段（不能在事件处理器内）。
- **Context 值若为原始值（数字/字符串），变更不触发更新**——用 `$state` 对象或函数 getter。
- **`$state.snapshot` 浏览器要求 98+ / Safari 15.4+**——兼容性发布时检查。
- **绑定 onMount 的 cleanup 顺序**：onMount 卸载时先调 cleanup，再调 onDestroy。

## FAQ

**Q: Svelte 5 还需要 stores 吗？**
A: 简单状态用 `$state` + `.svelte.js` 模块；stores 在复杂异步流、计时器、外部订阅、RxJS 互操作时仍有价值。

**Q: onMount vs $effect 怎么选？**
A: 需要访问 DOM 节点、启动副作用（fetch、定时器、绑定 window 事件）用 onMount；纯响应式派生/同步副作用用 `$effect`。

**Q: 为什么 onMount 的 async 函数清理不执行？**
A: `async () => {}` 总是返回 `Promise`；`onMount` 收到非函数就跳过清理。改用同步函数包异步逻辑：

```js
onMount(() => {
  let cancelled = false;
  (async () => {
    const data = await fetch(...);
    if (!cancelled) state = data;
  })();
  return () => { cancelled = true; };
});
```

**Q: Context 替代 props 的时机？**
A: 当一个值要穿透 3+ 层中间组件、或父组件不直接知道子组件（如 `{@render children()}`）时。

**Q: Vitest 单元测试和组件测试区别？**
A: 单元测试纯逻辑（runes 在 `.svelte.js` 文件中），无 DOM；组件测试用 jsdom 渲染完整 Svelte 组件。

**Q: IE 11 还能用 Svelte 5 吗？**
A: 不支持。最低 Chrome 87 / Firefox 83 / Safari 14。

**Q: SSR 项目用 `$state.snapshot` 安全吗？**
A: 服务端是 Node，无浏览器版本要求；仅客户端使用时检查例外表。

**Q: Playwright 跑测试前要 build 吗？**
A: `playwright.config.js` 中配 `webServer: { command: 'npm run build && npm run preview', port: 4173 }`，Playwright 自动起 preview 服务器。

## Examples

| 文件 | 内容 |
|---|---|
| `examples/lifecycle-hooks.md` | onMount 清理、onDestroy、tick DOM 测量、deprecated 钩子 |
| `examples/stores-advanced.md` | writable/readable/derived/自定义 store、异步流 |
| `examples/store-patterns.md` | stores vs `$state`、持久化、跨组件模式 |
| `examples/context-advanced.md` | createContext、setContext、与 state 组合、SSR 安全 |
| `examples/testing-vitest.md` | 单元/组件测试、$effect.root、flushSync、context wrapper |
| `examples/testing-playwright.md` | e2e 测试、Playwright config |

## References

| 文件 | 内容 |
|---|---|
| `references/lifecycle-hooks-reference.md` | onMount/onDestroy/tick 完整签名、SSR 注意事项 |
| `references/stores-api-reference.md` | 全部 store API、TypeScript 类型、契约 |
| `references/context-api-reference.md` | createContext/setContext/getContext/hasContext、SSR 行为 |
| `references/testing-reference.md` | Vitest/Storybook/Playwright 完整 setup |
| `references/browser-support.md` | 浏览器矩阵、例外特性、polyfill 指南 |
