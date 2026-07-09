---
name: svelte-misc
description: Svelte 5 杂项技能。当用户需要了解 TypeScript 支持、自定义元素、浏览器兼容性、Svelte 4→5 迁移、FAQ 时使用。
---

# Svelte Misc Reference (Svelte 5)

本技能覆盖 Svelte 5 的 TypeScript 高级支持、自定义元素（Web Components）、浏览器兼容性矩阵、Svelte 4→5 迁移指南、官方最佳实践与常见问答。

## When to use this skill

当用户需要 TypeScript 类型标注（泛型、包装组件、DOM 类型增强）、创建自定义 Web 组件并使用 `$host()` / `$bindable` / `extend`、查看浏览器兼容性矩阵与功能例外、从 Svelte 4 迁移到 Svelte 5，或遵循 Svelte 5 推荐的 `$state` / `$derived` / `$effect` 模式时使用本技能。

---

## Critical: TypeScript 高级

Svelte 原生支持 TypeScript（Vite 项目无需配置）。用 `<script lang="ts">` 启用类型注解；需要枚举 / 参数属性等运行时特性则需要预处理器。

```svelte
<script lang="ts">
  let name: string = 'world';
  function greet(name: string) { alert(`Hello, ${name}!`); }
</script>
<button onclick={(e: Event) => greet((e.target as HTMLElement).innerText)}>
  {name}
</button>
```

> `lang="ts"` 默认只支持「类型只消失」的特性（注解、接口、泛型）。运行时代码特性（enum、`private foo = 1` 形式的参数属性）需配置 `vitePreprocess({ script: true })`。

### 类型 Props + 透传

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';
  interface Props {
    requiredProperty: number;
    optionalProperty?: boolean;
    snippetWithStringArgument: Snippet<[string]>;
    eventHandler: (arg: string) => void;
    [key: string]: unknown; // 配合 ...rest 透传其他属性
  }
  let { requiredProperty, optionalProperty, snippetWithStringArgument,
        eventHandler, ...rest }: Props = $props();
</script>
```

### 泛型组件

`<script>` 的 `generics` 属性接受与函数泛型同样的语法：

```svelte
<script lang="ts" generics="Item extends { text: string }">
  interface Props { items: Item[]; select(item: Item): void; }
  let { items, select }: Props = $props();
</script>
{#each items as item}
  <button onclick={() => select(item)}>{item.text}</button>
{/each}
```

### 包装组件（转发原生属性）

```svelte
<script lang="ts">
  import type { HTMLButtonAttributes } from 'svelte/elements';
  let { children, ...rest }: HTMLButtonAttributes = $props();
</script>
<button {...rest}>{@render children?.()}</button>
```

无专用接口的元素用 `SvelteHTMLElements['div']` 形式。`Component` / `ComponentProps<T>` 用于约束动态组件与提取现有组件的 props。

### 类中 `$state`

```ts
class Counter {
  count = $state() as number; // 无初值类型会扩为 undefined
  constructor(initial: number) { this.count = initial; }
  increment() { this.count += 1; }
}
```

### 增强 `svelte/elements`

```ts
/// file: additional-svelte-typings.d.ts
declare module 'svelte/elements' {
  export interface SvelteHTMLElements { 'custom-button': HTMLButtonAttributes; }
  export interface HTMLAttributes<T> { globalattribute?: string; }
}
export {}; // 必须——否则会覆盖原模块
```

---

## Critical: Custom Elements（Web Components）

### 基础用法

```svelte
<svelte:options customElement="my-element" />
<script>
  let { name = 'world' } = $props();
</script>
<h1>Hello {name}!</h1>
<slot />
```

导入即自动 `customElements.define()` 注册。消费者可读写 DOM 属性：

```js
const el = document.querySelector('my-element');
el.name = 'everybody'; // 触发 shadow DOM 更新
```

> 必须显式声明 props（不能只写 `let props = $props()`），否则 Svelte 不知道要暴露哪些键。

### `customElement` 选项

```svelte
<svelte:options customElement={{
  tag: 'my-counter',
  shadow: 'open',
  props: { count: { type: 'Number', reflect: true, attribute: 'start' } },
  extend: (Ctor) => class extends Ctor { /* ... */ }
}} />
```

| 字段 | 取值 | 默认 | 作用 |
|------|------|------|------|
| `tag` | string | — | 标签名；设置后自动注册 |
| `shadow` | `'none' \| 'open' \| 'closed' \| ShadowRootInit` | `'open'` | Shadow root 配置 |
| `props.<name>.attribute` | string | 小写 prop 名 | 自定义属性名 |
| `props.<name>.reflect` | boolean | `false` | 将 prop 值镜像到 HTML 属性 |
| `props.<name>.type` | `'String' \| 'Boolean' \| 'Number' \| 'Array' \| 'Object'` | `'String'` | 属性 ↔ prop 类型转换 |
| `extend` | `(Ctor) => class` | — | 包装生成的类（可注入 lifecycle / ElementInternals）|

### `$host()` Rune + 生命周期

```svelte
<svelte:options customElement="tooltip" />
<script>
  import { $host } from 'svelte';
  let { text = 'hi' } = $props();
  function focus() { $host().focus(); }
</script>
<button onclick={focus}>{text}</button>
```

- 内部组件在 `connectedCallback` 之后的**下一个 tick** 创建；插入前设置的属性被缓冲并应用。
- Shadow DOM 更新批处理到下一个 tick；DOM 移动不会触发不必要的卸载。
- `disconnectedCallback` 后下一个 tick 销毁。组件方法 mount 后才可用；要更早暴露请在 `extend` 类里声明。

### 重要注意事项

- 样式**封装**而非「作用域」——全局 CSS 无法穿透 shadow root。
- CSS 内联为 JS 字符串而非单独 `.css` 文件。
- **不**适合 SSR（shadow DOM 在 JS 加载前不可见）。
- 插槽内容**积极渲染**（不像 Svelte 默认懒渲染），`<slot>` 在 `{#each}` 里不会复制内容。
- `let:` 指令对自定义元素**无效**。Svelte `context` 不能跨自定义元素边界。
- 不要命名 `on*` 属性（被解释为事件监听器）。

---

## Critical: Browser Support

Svelte 5 目标是 **Baseline 2020**。

| 浏览器 | 最低版本 |
|--------|----------|
| Chrome / Edge | 87 |
| Firefox | 83 |
| Safari | 14 |
| Opera | 73 |
| Opera (Android) | 62 |
| Samsung Internet | 14.0 |
| Android WebView | 87 |
| Internet Explorer | **不支持** |

### 例外情况

| 功能 | Chrome/Edge | Firefox | Safari |
|------|-------------|---------|--------|
| `$state.snapshot` | 98 | 94 | 15.4 |
| `bind:devicePixelContentBoxSize` | — | 93 | 不支持 |
| `flip`（来自 `svelte/animate`） | — | 126 | — |

### SSR 浏览器 API 保护

```svelte
<script>
  import { onMount } from 'svelte';
  onMount(() => {
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  });
</script>
```

> 优先用 `<svelte:window>` / `<svelte:document>`；Svelte 自动处理 SSR 与卸载。

---

## Critical: Best Practices

### `$state` — 只在需要时声明

只把会触发更新（`$effect` / `$derived` / 模板）的值声明为 `$state`。大型只重赋值对象（API 响应）用 `$state.raw` 避免 Proxy 开销：

```js
let user = $state.raw<User | null>(null);
```

### `$derived` > `$effect` 用于纯计算

```js
let square = $derived(num * num); // 好
// let square; $effect(() => { square = num * num; }); // 差
```

复杂逻辑用 `$derived.by(() => { ... })`。Derived 可写；对象/数组结果不会被深层代理。

### `$effect` 是 escape hatch

优先工具：同步外部库 → `{@attach ...}`；响应用户输入 → 处理器；调试日志 → `$inspect` / `$inspect.trace`；观察外部源 → `createSubscriber`。永远不要 `if (browser) { ... }` 包 effect——effect 本就不在服务端运行。

### Props 当作会变化、事件用属性语法

```js
let { type } = $props();
let color = $derived(type === 'danger' ? 'red' : 'green');
```

`<svelte:window>` / `<svelte:document>` 用于全局监听；不要 `onMount` 手动绑。Snippets 替代 Slots，`{#each ... (item.id)}` 用稳定 key（不可用 index）。`style:--` 把 JS 变量透到 CSS 自定义属性；用 CSS 变量（而非 `:global`）穿透子组件样式；用 `createContext` 替代模块级共享状态（提供类型安全 + 防止 SSR 用户间泄漏）。

### 新代码避免 Legacy

`on:click` → `onclick`；`export let` → `$props()`；`$:` → `$derived` / `$effect`；`<slot>` → snippet；`use:action` → `{@attach ...}`。

---

## Critical: Svelte 4 Migration Guide

### 最小版本

| 依赖 | 要求 |
|------|------|
| Node | 16+ |
| SvelteKit | 1.20.4+（如使用） |
| `vite-plugin-svelte` | 2.4.1+ |
| TypeScript | 5+ |

### 核心迁移对照

| Svelte 4 | Svelte 5 |
|---------|---------|
| `let x = 0`（顶层响应式） | `let x = $state(0)` |
| `$: x = a + b` | `let x = $derived(a + b)` |
| `$: { ... }` | `$effect(() => { ... })` |
| `export let prop` | `let { prop } = $props()` |
| `<slot />` | `{@render children()}` |
| `createEventDispatcher` | callback props |
| `on:click={fn}` | `onclick={fn}` |

### 自动迁移工具

```bash
npx svelte-migrate@latest svelte-5 my-project
```

### 重要行为变化

- 转换默认 `local`（外层控制流不影响子 transition）。
- 默认插槽绑定**不会**进入具名插槽。
- 预处理器顺序：markup → script → style（按声明顺序逐个执行）。
- `createEventDispatcher` 严格模式下检查可选 / 必传 / 无参类型（之后用 callback props 替代）。
- `Action` / `ActionReturn` 默认参数类型为 `undefined`，需要参数时显式声明。
- `SvelteComponentTyped` 已弃用，统一用 `SvelteComponent`。
- CJS 输出与 `svelte/register` 被移除。
- 删除 `svelte.JSX`，改用 `svelteHTML` / `svelte/elements`。
- 打包器必须指定 `browser` 条件（SvelteKit / Vite 自动；Rollup 用 `@rollup/plugin-node-resolve` 设 `browser: true`；Webpack 在 `conditionNames` 加 `"browser"`）。

---

## Critical: Svelte 5 Migration Guide (Concept Summary)

Runes 系统：`$state` / `$derived` / `$effect` 替代隐式响应式；Snippets 替代 Slots；事件属性替代 `on:`；`$effect` 显式追踪依赖；`$state` 对对象/数组做深层 Proxy。

| 移除 | 替代 |
|------|------|
| `let` 顶层隐式响应式 | `$state` |
| `$:` 语句 | `$derived` / `$effect` |
| `export let` / `$$props` / `$$restProps` | `$props()` |
| `createEventDispatcher` | callback props / `$bindable` |
| Slots | Snippets |
| `on:event` | `onevent` |
| `beforeUpdate` / `afterUpdate` | `$effect.pre` / `$effect` |

新增：`$state.eager`、`$effect.pre`、`$effect.tracking`、`$state.snapshot`、`hydratable`、`fork()`、`{#key}` 过渡触发、`<svelte:boundary>`、`createContext`。

---

## FAQ

**Q: Svelte 和 Vue/React 的核心区别？**
A: Svelte 无虚拟 DOM——编译时生成精确 DOM 更新代码，运行时零框架开销。

**Q: Svelte 5 和 Svelte 4 可以共存吗？**
A: 可以。Svelte 5 仍支持 Legacy Mode，Svelte 4 的 `$:` 与 `export let` 可继续用；可在同一项目渐进迁移。

**Q: 如何升级到 Svelte 5？**
A: ① 升级 `svelte` + `sveltekit` / `vite-plugin-svelte`；② 跑 `npx svelte-migrate@latest svelte-5`；③ 逐步迁到 Runes Mode。

**Q: Svelte 支持 SSR / TypeScript 吗？**
A: 都原生支持。SSR 用 `render()` 或 SvelteKit；TS 直接用 `<script lang="ts">`，CI 跑 `svelte-check`。

**Q: Svelte 有虚拟 DOM 吗？**
A: 没有。Svelte 编译器生成精确 DOM 操作代码（`element.textContent = ...`），而非通过 vDOM diffing。

**Q: 哪些浏览器支持 Svelte 5？**
A: Baseline 2020（Chrome / Edge 87+、Firefox 83+、Safari 14+）。不支持 IE；少数功能需更高版本（见上表）。

**Q: 自定义元素的样式如何控制？**
A: 默认 shadow root 模式 `'open'`，全局 CSS 不会穿透。可用 `shadow: 'none'` 继承全局样式；用 CSS 变量穿透 scoped 样式。

---

## Quick Fixes

| 问题 | 解决方案 |
|------|----------|
| Svelte 4 代码报错 | 确认组件未使用 `$state`（进入 Runes Mode） |
| `$:` 不生效 | 升级到 Svelte 5，Legacy Mode 仍支持 |
| TypeScript 报错 | 确认 `<script lang="ts">` 和 TypeScript 5+ |
| 迁移后打包变大 | 检查 tree-shaking 配置 |
| 属性以 `on` 开头被监听 | 重命名避免 `on*` 前缀 |
| 自定义元素 SSR 没内容 | 预渲染不支持 shadow DOM；等 JS 加载 |
| generic component 类型报错 | 检查 `generics` 字符串（与函数泛型同语法） |

---

## Gotchas

1. **Svelte 5 自动进入 Runes Mode** — 只要使用了任何 Rune（`$state` 等）即为 Runes Mode。
2. **Legacy Mode** — 可通过 `<svelte:options runes={false}>` 强制启用。
3. **`on:click` 已废弃** — Legacy Mode 仍可用，但会警告。
4. **`createEventDispatcher` 已弃用** — 新代码用 callback props。
5. **`$bindable(default)`** — 仅在父组件未传 prop 时作 fallback；父组件传 `undefined` 会保留 undefined。
6. **`bind:this` 泛型组件** — 实例类型为 `Component<Props> | null`，需要断言。
7. **自定义元素 inline CSS** — 没有可被外部样式表链接的 `.css` 文件。
8. **服务端 effect 不运行** — 不要用 `if (browser)` 包 effect 主体。

---

## Examples

Practical examples for Svelte 5 features.

- **Dynamic Components**: [svelte:element and {#key} blocks](./examples/dynamic-components.md)
- **Debug and HTML**: [@html, @debug, @const](./examples/debug-html.md)
- **TypeScript**:
  - [TypeScript Basics](./examples/typescript.md) — Typed props, generics, event handlers, component refs
  - [TypeScript Advanced](./examples/typescript-advanced.md) — 14 patterns: preprocessors, generic constraints, wrapper components, `Component` type, `$state` in classes, `svelte/elements` augmentation, `svelte-check`, `$bindable`, form state
- **Custom Elements**: [Custom Elements](./examples/custom-elements.md) — 12 patterns: `customElement` option, `$host()`, type coercion, reflection, `extend`, `ElementInternals`, slot semantics
- **Best Practices**: [Svelte 5 Best Practices](./examples/best-practices.md) — 14 patterns: `$state.raw`, `$derived.by`, keyed each blocks, CSS variables, `createContext`, async Svelte, legacy replacements
- **Migration**: [Svelte 3→4 Migration](./examples/migration-v4.md) — 10 patterns: `tag` → `customElement`, stricter types, transition scope, preprocessor order, CJS removal
- **Browser Support**: [Browser Support](./examples/browser-support.md) — 7 patterns: feature detection, polyfills, bundler target, SSR guards, `$state.snapshot` fallback, Playwright pinning

---

## References

In-depth reference documentation.

- **Dynamic Components**: [Dynamic Components Reference](./references/dynamic-reference.md) — `svelte:element`, `{#key}`, `{@html}`, `@debug`, `@const`
- **TypeScript**:
  - [TypeScript Basics](./references/typescript-reference.md) — Type inference, snippets, event types, generics
  - [TypeScript Advanced](./references/typescript-advanced.md) — `Component` type, `svelte/elements`, DOM type augmentation, `svelte-check`, errors
- **Custom Elements**: [Custom Elements](./references/custom-elements.md) — `customElement` surface, `$host()` rune, lifecycle, caveats, `ElementInternals`
- **Best Practices**: [Best Practices](./references/best-practices.md) — `$state` / `$derived` / `$effect`, props, events, snippets, each blocks, CSS variables, context, async
- **Browser Support**: [Browser Support](./references/browser-support.md) — Minimum versions matrix, feature exceptions, polyfills, SSR safety, bundler config

### Quick Links

| Feature | Reference |
|---------|-----------|
| `svelte:element` | [Dynamic Components](./references/dynamic-reference.md#svelteelement) |
| `{#key}` | [Dynamic Components](./references/dynamic-reference.md#key-block) |
| `{@html}` / `@debug` / `@const` | [Dynamic Components](./references/dynamic-reference.md) |
| TS basics | [TypeScript Basics](./references/typescript-reference.md) |
| TS advanced | [TypeScript Advanced](./references/typescript-advanced.md) |
| Generic components | [TypeScript Advanced](./references/typescript-advanced.md#generic-props) |
| Wrapper components | [TypeScript Advanced](./references/typescript-advanced.md#typing-wrapper-components) |
| `Component` type | [TypeScript Advanced](./references/typescript-advanced.md#the-component-type) |
| DOM augmentation | [TypeScript Advanced](./references/typescript-advanced.md#svelteelements--enhancing-built-in-dom-types) |
| `customElement` options | [Custom Elements](./references/custom-elements.md#customelement-option-surface) |
| `$host()` rune | [Custom Elements](./references/custom-elements.md#host-rune) |
| Web Component lifecycle | [Custom Elements](./references/custom-elements.md#component-lifecycle) |
| `$state` / `$derived` / `$effect` patterns | [Best Practices](./references/best-practices.md) |
| Async Svelte | [Best Practices](./references/best-practices.md#async-svelte-experimental) |
| Browser matrix | [Browser Support](./references/browser-support.md#minimum-browser-versions) |
| Feature exceptions | [Browser Support](./references/browser-support.md#feature-specific-exceptions) |
| Polyfills | [Browser Support](./references/browser-support.md#polyfills-for-older-browsers) |
