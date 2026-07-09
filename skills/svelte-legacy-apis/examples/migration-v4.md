# Svelte 3 → Svelte 4 Migration Examples

Practical, copy-pasteable patterns for upgrading a Svelte 3 project to Svelte 4. All examples reference categories from the official Svelte 4 migration guide.

> 自动化处理：`npx svelte-migrate@latest svelte-4` 会处理其中多数条目；其余需要手动介入。

---

## 1. Running the Migration Script

```bash
# Dry-run first
npx svelte-migrate@latest svelte-4

# Apply
npx svelte-migrate@latest svelte-4 ./src
```

脚本会自动处理：
- `tag=` → `customElement=`（`<svelte:options>`）
- `SvelteComponentTyped` → `SvelteComponent`
- `Action` / `ActionReturn` 泛型参数
- Transitions 默认行为（添加 `|global` 修饰符以保持旧行为）
- Slot 默认值（添加 `|local`）

---

## 2. Minimum Version Requirements

升级对应依赖版本：

```json
{
  "engines": { "node": ">=16" },
  "devDependencies": {
    "svelte": "^4.0.0",
    "svelte-loader": "^3.1.8",
    "rollup-plugin-svelte": "^7.1.5",
    "vite-plugin-svelte": "^2.4.1",
    "typescript": "^5.0.0"
  }
}
```

SvelteKit 用户额外：`@sveltejs/kit >= 1.20.4`。

---

## 3. Bundler Browser Conditions

`onMount` 等浏览器生命周期失效时，检查 bundler 是否声明了 `browser` 条件。

### Rollup (`@rollup/plugin-node-resolve`)

```js
// rollup.config.js
import { nodeResolve } from '@rollup/plugin-node-resolve';

export default {
  plugins: [
    nodeResolve({
      browser: true,             // ← 新增
      preferBuiltins: false,
      exportConditions: ['svelte', 'browser', 'default']  // 可选
    }),
    svelte({...})
  ]
};
```

### Webpack

```js
// webpack.config.js
module.exports = {
  resolve: {
    conditionNames: ['browser', 'import', 'require'],  // ← 加上 browser
    alias: {
      svelte: path.resolve('node_modules', 'svelte/src/runtime')
    }
  }
};
```

### 验证

```svelte
<script>
  import { onMount } from 'svelte';
  let mounted = false;
  onMount(() => { mounted = true; });
</script>
<p>{mounted ? 'SSR + CSR OK' : '等待 onMount'}</p>
```

如果 `mounted` 永远是 `false`，说明浏览器条件没启用。

---

## 4. Stricter Types for `createEventDispatcher`

### Svelte 3（可选 payload）

```ts
import { createEventDispatcher } from 'svelte';
const dispatch = createEventDispatcher<{
  optional: number | null;
  required: string;
  noArgument: null;
}>();

dispatch('optional');          // ok in v3
dispatch('required');          // 缺 detail 但 v3 允许
dispatch('noArgument', 'surprise');  // 多余参数但 v3 允许
```

### Svelte 4（严格）

```ts
import { createEventDispatcher } from 'svelte';
const dispatch = createEventDispatcher<{
  optional: number | null | undefined;  // undefined = 可选
  required: string;                     // 必填
  noArgument: null;                     // 不能传 detail
}>();

dispatch('optional');                  // OK
dispatch('required');                  // ❌ error TS2345
dispatch('noArgument', 'surprise');    // ❌ error TS2554
```

修复：
- 缺失 detail → `dispatch('required', '')` 或在类型上写 `required?: string`
- 多余 detail → 删除 `, 'surprise'`

---

## 5. `Action` / `ActionReturn` Generic Parameter

### Svelte 3（隐式推断）

```ts
import type { Action } from 'svelte';
// v3: params 类型从 Action 默认推断（any）
const copy: Action = (node, params) => {
  node.textContent = params.toUpperCase();  // params 是 any
};
```

### Svelte 4（必须显式泛型）

```ts
import type { Action } from 'svelte';

// v4: 必须显式给第二个泛型参数
const copy: Action<HTMLElement, string> = (node, params) => {
  node.textContent = params.toUpperCase();   // params: string
};

// 不需要参数
const log: Action<HTMLElement> = (node) => {
  console.log(node.tagName);
};
```

---

## 6. `onMount` Sync Cleanup Functions

### 反模式（v3 不会警告）

```js
onMount(async () => {
  const data = await fetch('/api');
  setup(data);
  return () => teardown();  // ← 永远不会被调用，因为返回 Promise<fn>，不是 fn
});
```

### Svelte 4 报错并修复

```js
onMount(() => {           // ← 必须同步
  let data;
  fetch('/api').then((d) => {
    data = d;
    setup(d);
  });
  return () => teardown();  // 现在会被调用
});
```

---

## 7. Custom Element `customElement` Option

### Svelte 3（`tag`）

```svelte
<svelte:options tag="my-counter" />

<script>
  export let count = 0;
</script>

<button on:click={() => count++}>{count}</button>
```

### Svelte 4（`customElement`，更可配置）

```svelte
<svelte:options customElement={{
  tag: 'my-counter',
  props: {
    count: { type: Number, attribute: 'count' }
  }
}} />

<script>
  let { count = 0 } = $props();
</script>
```

> 注意：Svelte 4 中 `<svelte:options customElement>` 仍可工作，但若该组件最终要迁移到 Svelte 5，把 props 类型写成对象形式更稳。

---

## 8. `SvelteComponentTyped` → `SvelteComponent`

```diff
- import { SvelteComponentTyped } from 'svelte';
+ import { SvelteComponent } from 'svelte';

- export class Foo extends SvelteComponentTyped<{ aProp: string }> {}
+ export class Foo extends SvelteComponent<{ aProp: string }> {}
```

`typeof SvelteComponent` 在动态组件场景显式参数化：

```svelte
<script>
  import ComponentA from './ComponentA.svelte';
  import ComponentB from './ComponentB.svelte';
  import { SvelteComponent } from 'svelte';

  let component: typeof SvelteComponent<any>;  // ← Svelte 4 需要 <any>

  function pick() {
    component = Math.random() > 0.5 ? ComponentA : ComponentB;
  }
</script>
<button on:click={pick}>pick</button>
<svelte:element this={component} />
```

---

## 9. Transitions Are Local by Default

### 旧行为（v3 跨控制块触发）

```svelte
{#if show}
  {#if success}
    <p transition:slide>OK</p>
  {/if}
{/if}
```
v3 中，`show` 从 false → true 会触发 `slide` 进入。

### 新行为（v4 只在直接父块触发）

```svelte
{#if show}
  {#if success}
    <p transition:slide|global>OK</p>  <!-- 显式声明为全局 -->
  {/if}
{/if}
```

迁移脚本会为受影响的 transitions 添加 `|global`。

---

## 10. Default Slot Bindings Not Exposed to Named Slots

```svelte
<!-- Nested.svelte -->
<script>
  export let count = 0;
</script>
<slot />
<slot name="bar" />
```

```svelte
<!-- Parent.svelte -->
<script>
  import Nested from './Nested.svelte';
</script>

<Nested let:count>
  <p>default 可见：{count}</p>
  <p slot="bar">bar 不可见：{count}</p>  <!-- ❌ count undefined -->
</Nested>
```

修复：在 named slot 里显式传值：

```svelte
<!-- Nested.svelte -->
<slot name="bar" {count} />

<!-- Parent.svelte -->
<Nested let:count>
  <p>default 可见：{count}</p>
  <p slot="bar" let:count>bar 可见：{count}</p>
</Nested>
```

---

## 11. Preprocessor Order Changes

Svelte 4 改为按声明顺序执行，且各 preprocessor 必须有 `name`。

```js
// svelte.config.js
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';
import { mdsvex } from 'mdsvex';

export default {
  preprocess: [
    mdsvex({ name: 'mdsvex' }),     // 先 markup，再 script
    vitePreprocess({ name: 'vite' }) // 后 markup
  ]
};
```

v3 会先所有 markup 再所有 script；v4 严格顺序：每个 preprocessor 内部 markup → script → style，然后下一个。

每个 preprocessor 必须有 name：

```js
const myPreprocess = {
  name: 'my-preprocess',
  markup: ({ content }) => ({ code: content.replace(/X/g, 'Y') })
};
```

---

## 12. Other Notable v4 Changes

- `inert` 自动应用到 outroing 元素（无障碍改进）
- `classList.toggle(name, boolean)` —— 超老浏览器需 polyfill
- `CustomEvent` 构造函数 —— 超老浏览器需 polyfill
- 自定义 store 的 `StartStopNotifier` 现在需要传入 update 函数
- `derived(store)` 在 falsy 值上抛错（之前仅警告）
- `svelte/internal` 类型定义移除 —— 不再是公开 API
- DOM 节点 removal 现在是 batched，`MutationObserver` 顺序可能变化
- `svelte.JSX` 命名空间增强 → 用 `svelteHTML`；读 `svelte/elements` 替代

---

## Run-it-Yourself Checklist

```bash
# 1. 升级依赖
npm install svelte@^4

# 2. 跑迁移脚本
npx svelte-migrate@latest svelte-4

# 3. 看 TypeScript
npx svelte-check

# 4. 跑测试
npm test
```

如果有大量 `Action` 泛型错误，可能是 migration 脚本漏掉；手动补 `<HTMLElement, YourType>` 即可。
