# Svelte 3 → Svelte 4 Migration Reference

完整对照参考，基于官方 Svelte 4 migration guide 整理。每个条目包含：变更、影响范围、修复方式。

---

## Table of Contents

1. Minimum version requirements
2. Browser conditions for bundlers
3. Removal of CJS output
4. Stricter types for Svelte functions
5. Custom Elements overhaul
6. `SvelteComponentTyped` is deprecated
7. Transitions are local by default
8. Default slot bindings
9. Preprocessor order changes
10. New eslint package
11. Other breaking changes
12. Library author guidance

---

## 1. Minimum Version Requirements

| 工具 | 最低版本 | 原因 |
|------|---------|------|
| Node.js | 16 | 运行时 |
| SvelteKit | 1.20.4 | 配套修复 |
| vite-plugin-svelte | 2.4.1 | browser condition |
| webpack | 5 | ESM 支持 |
| svelte-loader | 3.1.8 | webpack 5 配套 |
| rollup-plugin-svelte | 7.1.5 | 同样原因 |
| TypeScript | 5 | 类型严格化 |

---

## 2. Browser Conditions for Bundlers

**症状**：`onMount`、`beforeNavigate` 等在浏览器中不调用。

**修复**：bundler 必须声明 `browser` export condition。

### Vite / SvelteKit
无需操作 —— 已自动处理。

### Rollup

```js
// rollup.config.js
import { nodeResolve } from '@rollup/plugin-node-resolve';

nodeResolve({
  browser: true,                       // ← 关键
  exportConditions: ['svelte', 'browser']
});
```

### Webpack

```js
// webpack.config.js
module.exports = {
  resolve: {
    conditionNames: ['browser', 'import', 'require', 'default']
  }
};
```

---

## 3. Removal of CJS Output

**移除**：
- 编译器的 CJS 输出
- `svelte/register` hook
- CJS runtime

**需要 CJS**：使用 bundler 把 Svelte 的 ESM 输出转 CJS。

---

## 4. Stricter Types for Svelte Functions

### `createEventDispatcher`

| 字段类型标记 | v3 行为 | v4 行为 |
|-------------|---------|---------|
| `T \| null` | 可选 | 可选（OK） |
| `T` | 可缺 detail | 必须给 detail |
| `null` | 可多带 detail | 不允许 detail |
| `T \| undefined` | 可选 | **可选（OK）** |

### `Action` / `ActionReturn`

第二个泛型参数现在默认是 `undefined`：

```ts
// v3
const action: Action = (node, params) => {...}     // params: any

// v4
const action: Action<HTMLElement, string> = (node, params) => {...}
```

### `onMount`

异步返回的清理函数不再被调用 —— 现在 TS 会报错。

```js
// 反例（v4 报错）
onMount(async () => {
  await setup();
  return () => cleanup();   // ❌ 不会清理
});

// 正例
onMount(() => {
  setup();
  return () => cleanup();
});
```

---

## 5. Custom Elements Overhaul

| v3 | v4 |
|----|----|
| `<svelte:options tag="x" />` | `<svelte:options customElement="x" />` |
| 仅字符串 | 字符串 或 `{ tag, props, observedAttributes }` |

```svelte
<!-- v4 -->
<svelte:options customElement={{
  tag: 'my-counter',
  props: {
    initial: { type: Number, attribute: 'initial' },
    label: { type: String }
  }
}} />
```

**注意**：prop 更新时机略有变化（从被动转主动）。

---

## 6. `SvelteComponentTyped` Deprecated

合并到 `SvelteComponent`。`typeof SvelteComponent` 需要 `<any>` 显式参数化：

```diff
- import { SvelteComponentTyped } from 'svelte';
+ import { SvelteComponent } from 'svelte';

- export class Foo extends SvelteComponentTyped<{ a: string }> {}
+ export class Foo extends SvelteComponent<{ a: string }> {}

- let component: typeof SvelteComponent;
+ let component: typeof SvelteComponent<any>;
```

迁移脚本会自动处理。

---

## 7. Transitions Are Local by Default

| 范围 | 含义 |
|------|------|
| local（默认） | 仅直接父块 created/destroyed 触发 |
| global | 所有祖先块 created/destroyed 都触发 |

```svelte
{#if show}
  {#if success}
    <!-- local：只在 success 切换时触发 -->
    <p transition:slide>OK</p>

    <!-- global：show 切换也触发 -->
    <p transition:slide|global>OK</p>
  {/if}
{/if}
```

---

## 8. Default Slot Bindings Not Exposed to Named Slots

```svelte
<!-- Component.svelte -->
<slot {count} />
<slot name="bar" />
```

```svelte
<!-- 在 v3：default 的 let:count 也对 bar 可见（混乱） -->
<!-- 在 v4：named slot 看不到 default 的 let 绑定 -->
<Component let:count>
  <p>default 可见：{count}</p>
  <p slot="bar">bar 不可见：{count}</p>  <!-- ❌ -->
</Component>
```

---

## 9. Preprocessor Order Changes

v3：`markup-1, markup-2, ..., script-1, script-2, ..., style-1, style-2, ...`
v4：`markup-1, script-1, style-1, markup-2, script-2, style-2, ...`

**额外要求**：每个 preprocessor 必须有 `name` 属性。

```js
const myPreprocess = {
  name: 'my-preprocess',
  markup: ({ content }) => ({ code: content }),
  script: ({ content }) => ({ code: content }),
  style: ({ content }) => ({ code: content })
};
```

`MDsveX` 用户：必须放在最前（在 vitePreprocess 之前）。

---

## 10. New ESLint Package

| 旧 | 新 |
|----|----|
| `eslint-plugin-svelte3` | `eslint-plugin-svelte` |

迁移方式：
- `npm create svelte@latest` 创建新项目选 ESLint
- 拷贝 `.eslintrc*` 与 `eslint.config.*`

---

## 11. Other Breaking Changes

| 影响 | 说明 |
|------|------|
| `inert` 属性 | outroing 元素默认 `inert`（无障碍） |
| `classList.toggle(name, boolean)` | 超老浏览器需要 polyfill |
| `CustomEvent` 构造函数 | 超老浏览器需要 polyfill |
| 自定义 store `StartStopNotifier` | 现在需要传 update 函数 |
| `derived` 抛错 | falsy 值或非 store 现在报错 |
| `svelte/internal` 类型 | 已移除（非公开 API） |
| DOM 节点移除 | 现在是 batched |
| `svelte.JSX` 增强 | 改用 `svelteHTML` / 读 `svelte/elements` |
| `typeof SvelteComponent` | 现在需要显式 `<any>` 泛型参数 |

---

## 12. Library Author Guidance

决定是否同时支持 Svelte 3 与 Svelte 4：

```jsonc
{
  "peerDependencies": {
    "svelte": "^3.0.0 || ^4.0.0"
  },
  "devDependencies": {
    "svelte": "^4.0.0"
  }
}
```

多数 v4 破坏性变更不影响库作者，主要影响 app：

- bundler browser condition —— 由应用 bundler 处理
- 严格类型 —— 需要更新 TS，但运行时仍兼容
- `tag` → `customElement` —— 影响自动迁移，但运行时兼容

### 自动化迁移能处理的

- `tag=` → `customElement=` (string form)
- `SvelteComponentTyped` → `SvelteComponent`
- `Action` / `ActionReturn` 泛型参数
- Transitions 默认行为（添加 `|global`）

### 需要手动处理的

- Bundler `browser` condition 配置
- 严格类型错误的修复
- `onMount` 异步清理改同步
- Preprocessor name 属性
- ESLint 插件替换

---

## Related Source Material

- 官方 Svelte 4 migration guide: `/tmp/svelte-llms.txt` line 6182 onwards
- 对应 v5 migration guide: line 6427 onwards
- 详细示例：[migration-v4.md](../examples/migration-v4.md)
- 现有 v4→v5 对照：[migration-reference.md](./migration-reference.md)
