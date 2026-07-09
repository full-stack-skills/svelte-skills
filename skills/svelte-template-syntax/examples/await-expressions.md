# await Expressions（experimental）

Svelte 5.36+ 在模板中支持 `await` 表达式。需在 `svelte.config.js` 启用 `experimental.async`。

## 0. 启用

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

可在三个位置使用 `await`：
- `<script>` 顶层
- `$derived(...)` 内
- 模板内

## 1. 模板顶层 await

```svelte
<script>
  async function loadUser(id) {
    const res = await fetch(`/api/users/${id}`);
    return res.json();
  }

  let id = $state(1);
</script>

<input bind:value={id} />
<p>Name: {await loadUser(id)}</p>
```

## 2. 同步更新（synchronized）

当 `await` 依赖某 state，state 变化直到异步完成才反映到 UI，避免不一致状态。

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

如果增加 `a`，`<p>` 不会立刻变成 `2 + 2 = 3` — 等待 `add(a, b)` resolve 后再更新成 `2 + 2 = 4`。

**快速更新会重叠**：慢更新还没完成时，快更新可先反映到 UI。

## 3. 并发（concurrency）

独立 `await` 表达式并行运行：

```svelte
<p>{await one(x)}</p>
<p>{await two(y)}</p>
```

视觉上顺序执行，实际并行 — 它们是独立表达式。

**例外**：在 `<script>` 内的 async 函数里，顺序 `await` 是串行的。

```js
// 串行（waterfall）— 会触发 await_waterfall 警告
let a = $derived(await one(x));
let b = $derived(await two(y));
// a 创建后才会创建 b，但稳定后两者独立更新
```

## 4. 加载状态指示

用 `<svelte:boundary>` 的 `pending` snippet 包装：

```svelte
<script>
  async function delayed(s) {
    await new Promise(f => setTimeout(f, 1000));
    return s;
  }
</script>

<svelte:boundary>
  <p>{await delayed('hello!')}</p>

  {#snippet pending()}
    <p>loading...</p>
  {/snippet}
</svelte:boundary>
```

> `pending` 只在 boundary 第一次创建时显示，**不**在后续异步更新时显示。

## 5. $effect.pending()

后续异步工作时用 `$effect.pending()` 检测：

```svelte
<script>
  let valid = $state(false);
  let validating = $state(false);

  $effect(() => {
    if (!valid) return;
    validating = true;
    (async () => {
      await fetch(`/api/check?q=${query}`);
      validating = false;
    })();
  });
</script>

<input bind:value={query} oninput={() => valid = true} />
{#if validating}<span>checking...</span>{/if}
```

> 实际用法：在 effect 内调用 `$.effect.pending()`。

## 6. settled()

`settled()` 返回当前更新完成时 resolve 的 Promise：

```svelte
<script>
  import { tick, settled } from 'svelte';
  let color = 'red';
  let updating = false;

  async function onclick() {
    updating = true;
    await tick(); // 让 updating = true 先渲染
    color = 'octarine';
    await settled(); // 等待所有受 color 影响的更新应用
    updating = false;
  }
</script>
```

## 7. 错误处理

`await` 抛出的错误冒泡到最近的 `<svelte:boundary>`：

```svelte
<svelte:boundary>
  <Profile userId={id} />

  {#snippet failed(error, reset)}
    <button onclick={reset}>retry</button>
  {/snippet}
</svelte:boundary>
```

> 事件处理器、setTimeout、async 函数内的错误**不**被 boundary 捕获。

## 8. Forking（5.42+）

`fork(...)` 用于预执行期望很快发生的异步（如 SvelteKit 的预加载）：

```svelte
<script>
  import { fork } from 'svelte';
  import Menu from './Menu.svelte';

  let open = $state(false);
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

{#if open}
  <Menu onclose={() => open = false} />
{/if}
```

## 9. SSR

`render(...)` 现在是异步的：

```js
// server.js
import { render } from 'svelte/server';
import App from './App.svelte';

const { head, body } = await render(App);
```

带 `pending` snippet 的 boundary 在 SSR 时渲染 pending 内容；其他 `await` 在 `await render(...)` 返回前 resolve。

## 10. 注意事项

- 启用 `experimental.async` 后 effect 顺序略有变化：`{#if}` / `{#each}` block effects 在 `$effect.pre` / `beforeUpdate` 之前运行
- API（如 `$effect.pending()`）在 6.0 之前可能调整
- 已废弃的 `{#await ...}` 块仍然可用，但有性能开销（每次分支切换会创建/销毁 DOM）

## 11. {#await} vs await 表达式

| 场景 | 推荐 |
|------|------|
| 旧代码、SSR 简单分支 | `{#await ...}` 块 |
| 新代码、需要同步更新 | `await` 表达式 |

```svelte
<!-- 旧：{#await} 块（仍然支持） -->
{#await promise}
  <p>loading</p>
{:then value}
  <p>{value}</p>
{/await}

<!-- 新：await 表达式（experimental） -->
<svelte:boundary>
  <p>{await promise}</p>
  {#snippet pending()}<p>loading</p>{/snippet}
</svelte:boundary>
```
