# Browser Support Reference

## 基线目标

Svelte 5 自身的浏览器基线目标是 **2020 (Baseline 2020)**，即下表所示的最低版本：

| 浏览器 | 最低版本 |
|---|---|
| Chrome / Edge | 87 |
| Firefox | 83 |
| Safari | 14 |
| Opera | 73 |
| Opera (Android) | 62 |
| Samsung Internet | 14.0 |
| Android WebView | 87 |
| Internet Explorer | **不支持** |

> 上表基于 Svelte 内部使用的浏览器 API。**不包含** SvelteKit、第三方库或你自己的代码。

## 例外特性（需要更高版本）

| 特性 | Chrome/Edge | Firefox | Safari |
|---|---|---|---|
| `$state.snapshot` | 98 | 94 | 15.4 |
| `bind:devicePixelContentBoxSize` | — | 93 | 不支持 |
| `flip` from `svelte/animate` | — | 126 | — |

> 仅在使用这些特性时才需要关注。

## 检查清单

发布前需要回答：

- [ ] 目标用户是否覆盖 Chrome 87 以下？——需 polyfill 或放弃
- [ ] 目标用户是否覆盖 Firefox 83 以下？——需 polyfill 或放弃
- [ ] Safari 14 是否覆盖？——iOS 14+ 即可
- [ ] 是否使用 `$state.snapshot`？——若支持旧 Safari 则需替代方案
- [ ] 是否使用 `bind:devicePixelContentBoxSize`？——Firefox < 93 不支持
- [ ] 是否使用 `flip` 动画？——Firefox < 126 不支持

## Polyfill 策略

Svelte 5 内部使用：

- **Proxy**（ES2015）：所有现代浏览器已支持
- **Object.assign / Spread**：ES2018+
- **async/await**：ES2017+
- **Top-level await**：ES2022+
- **Optional chaining / nullish coalescing**：ES2020+

这些都在 Baseline 2020 范围内，**不需要 polyfill**。

## SSR 运行时要求

| 运行时 | 最低版本 |
|---|---|
| Node.js | 18+ |
| Bun | 1.0+ |
| Deno | 1.30+ |

## 模块解析

Svelte 5 依赖 `browser` 条件。Vite/SvelteKit 自动配置；自定义打包器需手动设置：

### Rollup

```js
// rollup.config.js
import { nodeResolve } from '@rollup/plugin-node-resolve';
export default {
  plugins: [nodeResolve({ browser: true })]
};
```

### Webpack

```js
// webpack.config.js
module.exports = {
  resolve: {
    conditionNames: ['browser', 'import', 'default']
  }
};
```

> 没有 `browser` 条件可能导致 `onMount` 等生命周期钩子不执行。

## 测试浏览器兼容性

### Playwright 项目配置

```js
// playwright.config.js
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    { name: 'chrome-87', use: { ...devices['Desktop Chrome'], launchOptions: { executablePath: '...' } } },
    { name: 'firefox-83', use: { ...devices['Desktop Firefox'] } },
    { name: 'safari-14', use: { ...devices['Desktop Safari'] } }
  ]
});
```

### Browserslist

```json
// .browserslistrc
> 0.5%
last 2 versions
not dead
not IE 11
not op_mini all
```

## 已知浏览器陷阱

- **Safari 14**：`Intl` API 部分实现；时区/日期处理需谨慎
- **Firefox 83**：`ResizeObserver` 已支持；`IntersectionObserver` 已支持
- **Chrome 87**：`top-level await` 不可用（87 = 89 才有）

## Svelte 5 在 SSR / Node 上的浏览器

SSR 不涉及浏览器 API。**所有 Svelte 5 SSR 兼容性仅受 Node 运行时版本约束**。
