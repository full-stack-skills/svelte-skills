# await Expressions Reference

Svelte 5.36+ 在 `<script>` 顶层、`$derived(...)` 内、模板内支持 `await` 表达式（需 `experimental.async`）。

## 启用

```js
// svelte.config.js
export default {
  compilerOptions: {
    experimental: {
      async: true
    }
  }
};
```

`experimental.async` 标志将在 Svelte 6.0 移除。

## 三个允许位置

1. `<script>` 顶层
2. `$derived(...)` 内
3. 模板内

## 同步更新（Synchronized）

`await` 表达式依赖某 state，state 变化直到异步完成才反映到 UI — 避免不一致中间状态。

```svelte
<script>
  let a = $state(1);
  let b = $state(2);
  async function add(a, b) {
    await new Promise(f => setTimeout(f, 500));
    return a + b;
  }
</script>

<input type="number" bind:value={a} />
<input type="number" bind:value={b} />
<p>{a} + {b} = {await add(a, b)}</p>
```

增加 `a` 时，`<p>` 不会立刻变成 `2 + 2 = 3` — 等待 `add(a, b)` resolve 后才更新到 `2 + 2 = 4`。

**快速更新可重叠**：慢更新未完成时，快更新可优先反映到 UI。

## 并发（Concurrency）

独立 `await` 表达式**并行**运行（即使视觉上顺序）：

```svelte
<p>{await one(x)}</p>
<p>{await two(y)}</p>
```

**例外**：在 `<script>` 内或 async 函数中，顺序 `await` 串行执行。

```js
// 串行（waterfall）
let a = $derived(await one(x));
let b = $derived(await two(y));
// 初始 a 创建完成后才创建 b
// 稳定后两者独立更新
```

> 会触发 `await_waterfall` 运行时警告。

## 加载状态

用 `<svelte:boundary>` + `pending` snippet 包装：

```svelte
<svelte:boundary>
  <p>{await delayed('hello')}</p>
  {#snippet pending()}
    <p>loading...</p>
  {/snippet}
</svelte:boundary>
```

> `pending` 只在 boundary **第一次**创建时显示；后续异步更新用 `$effect.pending()`。

## $effect.pending()

后续异步工作时检测：

```js
$effect(() => {
  // 异步工作时调用 $.effect.pending() 报告
  // 注意：实际 API 名称在不同 Svelte 5.x 版本可能不同
});
```

常用于：表单输入的 "checking..." spinner。

## settled()

返回当前所有更新完成时 resolve 的 Promise：

```svelte
<script>
  import { tick, settled } from 'svelte';

  let color = 'red';
  let updating = false;

  async function onclick() {
    updating = true;
    await tick();          // 让 updating = true 先渲染
    color = 'octarine';
    await settled();       // 等待所有受 color 影响的更新应用
    updating = false;
  }
</script>
```

## 错误处理

`await` 抛出的错误冒泡到**最近的** `<svelte:boundary>`：

```svelte
<svelte:boundary>
  <Profile userId={id} />

  {#snippet failed(error, reset)}
    <button onclick={reset}>retry</button>
  {/snippet}
</svelte:boundary>
```

> 事件处理器、`setTimeout`、async 函数**内**的错误**不**被 boundary 捕获。

## SSR

`render(...)` 现在是异步：

```js
import { render } from 'svelte/server';
import App from './App.svelte';

const { head, body } = await render(App);
```

> SvelteKit 等框架自动处理。

带 `pending` snippet 的 boundary 在 SSR 时渲染 pending 内容；其他 `await` 在 `await render(...)` 返回前 resolve。

未来计划：streaming SSR（后台渲染内容）。

## Forking（5.42+）

`fork(...)` API 用于**预执行**期望很快发生的异步（SvelteKit 的预加载场景）：

```svelte
<script>
  import { fork } from 'svelte';

  /** @type {import('svelte').Fork | null} */
  let pending = null;

  function preload() {
    pending ??= fork(() => {
      open = true;
    });
  }

  function discard() {
    pending?.discard();
    pending = null;
  }
</script>

<button
  onfocusin={preload}
  onfocusout={discard}
  onpointerenter={preload}
  onpointerleave={discard}
  onclick={() => {
    pending?.commit();
    pending = null;
    open = true;
  }}
>open menu</button>
```

`fork` 返回的对象：
- `discard()` — 取消
- `commit()` — 提交执行

## Effect 顺序变化

启用 `experimental.async` 后：
- `{#if}` / `{#each}` block effects 在 `$effect.pre` / `beforeUpdate` 之前运行
- 极少情况下会改变行为

## 限制与注意事项

- 实验性 API — 6.0 前可能调整
- `$effect.pending()` 实际 API 形式以官方文档为准
- 已废弃的 `{#await ...}` 块仍可用，但有性能开销
- 串行 `$derived(await ...)` 会触发 `await_waterfall` 警告
- 不在 boundary 内的 `await` 抛错会让组件崩溃

## 模式对照

| 模式 | 旧（`{#await}` 块） | 新（await 表达式） |
|------|---------------------|---------------------|
| 加载 | `{#await promise}loading{:then v}{v}{/await}` | `<svelte:boundary>{await p}{#snippet pending()}loading{/snippet}</svelte:boundary>` |
| 错误 | `{:catch e}{e.message}` | `<svelte:boundary>{#snippet failed(e, reset)}…{/snippet}` |
| 同步 | ❌ | ✅ |
| 嵌套 | OK | 配合 boundary |
| SSR | 简单 | 异步 render |

## 何时用哪个

- **新代码**：用 await 表达式 + boundary
- **简单分支 + 旧版兼容**：`{#await}` 块
- **需要 cancellation**：fork + abort controller
- **流式 SSR**：用边界（未来支持）
