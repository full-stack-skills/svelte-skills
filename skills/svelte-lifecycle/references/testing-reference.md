# Testing Reference

## Vitest Setup

### 安装

```sh
npm install -D vitest jsdom
```

### vite.config.js

```js
// vite.config.js
import { defineConfig } from 'vitest/config';
import { svelte } from '@sveltejs/vite-plugin-svelte';

export default defineConfig({
  plugins: [svelte({ hot: false })],
  test: {
    environment: 'jsdom',
    include: ['**/*.test.{js,ts}', '**/*.svelte.test.{js,ts}'],
    globals: false
  },
  resolve: process.env.VITEST
    ? { conditions: ['browser'] }
    : undefined
});
```

> `resolve.conditions: ['browser']` 让 Vitest 在 Node 中运行时也能使用 Svelte 的 browser 入口。

### 单文件环境声明

```js
// @vitest-environment jsdom
import { mount } from 'svelte';
```

### Runes 在测试中

文件名包含 `.svelte.` 时 Vitest 让 runes 可用：

```js
// counter.svelte.test.js
import { flushSync } from 'svelte';
import { expect, test } from 'vitest';

test('Counter', () => {
  let count = $state(0);
  count = 5;
  flushSync();
  expect(count).toBe(5);
});
```

## 组件测试 API

### mount / unmount / flushSync

```js
import { mount, unmount, flushSync } from 'svelte';
```

| 函数 | 用途 |
|---|---|
| `mount(Component, { target, props })` | 渲染组件到 target，返回 component 实例 |
| `unmount(component)` | 卸载 |
| `flushSync()` | 强制同步运行所有 pending effects（测试必备） |

> `mount` 不会自动运行 `onMount` 和 `$effect` —— 测试中需要 `flushSync()`。

### 基础组件测试

```js
import { mount, unmount, flushSync } from 'svelte';
import { expect, test } from 'vitest';
import Counter from './Counter.svelte';

test('Counter', () => {
  const c = mount(Counter, { target: document.body, props: { n: 0 } });
  expect(document.body.innerHTML).toBe('<button>0</button>');
  document.body.querySelector('button').click();
  flushSync();
  expect(document.body.innerHTML).toBe('<button>1</button>');
  unmount(c);
});
```

### @testing-library/svelte

```sh
npm install -D @testing-library/svelte @testing-library/user-event @testing-library/jest-dom
```

```js
import { render, screen } from '@testing-library/svelte';
import userEvent from '@testing-library/user-event';
import { expect, test } from 'vitest';
import Counter from './Counter.svelte';

test('Counter', async () => {
  const user = userEvent.setup();
  render(Counter);
  const btn = screen.getByRole('button');
  expect(btn).toHaveTextContent('0');
  await user.click(btn);
  expect(btn).toHaveTextContent('1');
});
```

### 复杂 props / context 包装器

```js
function Wrapper(...args) {
  setUser({ name: 'Bob' });
  return MyComponent(...args);
}

mount(Wrapper, { target: document.body });
```

## $effect.root

测试中使用 `$effect` 必须包在 `$effect.root` 内：

```js
import { flushSync } from 'svelte';
import { expect, test } from 'vitest';

test('Effect', () => {
  const cleanup = $effect.root(() => {
    let count = $state(0);
    const log = [];
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

## Storybook Setup

### 通过 Svelte CLI

```sh
npx sv add storybook
```

选择包含 testing 功能的推荐配置。

### 手动安装

```sh
npx storybook@latest init
```

### Story 示例

```svelte
<!-- Button.stories.svelte -->
<script module>
  import { defineMeta } from '@storybook/addon-svelte-csf';
  import { expect, fn } from 'storybook/test';
  import Button from './Button.svelte';

  const { Story } = defineMeta({
    component: Button,
    args: { onclick: fn() }
  });
</script>

<Story name="Primary" args={{ label: 'Click' }} />

<Story
  name="Click Test"
  play={async ({ canvas, userEvent, args }) => {
    await userEvent.click(canvas.getByRole('button'));
    await expect(args.onclick).toHaveBeenCalledTimes(1);
  }}
/>
```

## Playwright Setup

### 安装

```sh
npm install -D @playwright/test
npx playwright install
```

### playwright.config.js

```js
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: 'tests',
  testMatch: /(.+\.)?(test|spec)\.[jt]s/,
  use: { baseURL: 'http://localhost:4173' },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } }
  ],
  webServer: {
    command: 'npm run build && npm run preview',
    port: 4173,
    reuseExistingServer: !process.env.CI
  }
});
```

### 测试

```js
import { expect, test } from '@playwright/test';

test('h1 visible', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('h1')).toBeVisible();
});
```

## 工具对比

| 工具 | 用途 | 速度 | 真实度 |
|---|---|---|---|
| Vitest (单元) | 测试纯逻辑、runes | 极快 | 低（无 DOM） |
| Vitest + jsdom (组件) | 组件渲染 + 交互 | 快 | 中（jsdom 模拟） |
| Storybook | UI 文档 + 交互测试 | 中 | 高（真实浏览器） |
| Playwright | 端到端 | 慢 | 最高 |

## 决策指南

- **先写单元测试**：将业务逻辑提取到 `.svelte.js` 文件，直接测
- **需要测组件行为**：用 Vitest + jsdom 或 testing-library
- **需要视觉回归 / 多状态覆盖**：用 Storybook + 视觉快照
- **需要测完整用户流程**：用 Playwright
