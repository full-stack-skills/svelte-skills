# End-to-End Testing with Playwright

## 1. 基础 e2e 测试

```js
// tests/home.spec.js
import { expect, test } from '@playwright/test';

test('home page has h1', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('h1')).toBeVisible();
});
```

## 2. 点击 + 断言

```js
// tests/counter.spec.js
import { expect, test } from '@playwright/test';

test('counter increments', async ({ page }) => {
  await page.goto('/counter');
  const btn = page.getByRole('button');
  await expect(btn).toHaveText('0');
  await btn.click();
  await expect(btn).toHaveText('1');
});
```

## 3. 表单填写

```js
// tests/login.spec.js
import { expect, test } from '@playwright/test';

test('user can log in', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('ada@example.com');
  await page.getByLabel('Password').fill('secret');
  await page.getByRole('button', { name: /log in/i }).click();
  await expect(page).toHaveURL('/dashboard');
  await expect(page.getByText('Welcome, Ada')).toBeVisible();
});
```

## 4. 等待 store 异步更新

```js
test('loads posts after fetch', async ({ page }) => {
  await page.goto('/posts');
  await expect(page.getByText('Loading…')).toBeVisible();
  await expect(page.getByRole('article')).toHaveCount(5);
});
```

## 5. 跨页 navigation + Context 状态保留

```js
test('theme persists across pages', async ({ page }) => {
  await page.goto('/');
  await page.getByRole('button', { name: 'Toggle theme' }).click();

  await page.goto('/about');
  await expect(page.locator('html')).toHaveClass(/dark/);
});
```

## 6. 键盘交互

```js
test('keyboard shortcut opens search', async ({ page }) => {
  await page.goto('/');
  await page.keyboard.press('Control+K');
  await expect(page.getByPlaceholder('Search…')).toBeFocused();
});
```

## 7. playwright.config.js 完整配置

```js
// playwright.config.js
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: 'tests',
  testMatch: /(.+\.)?(test|spec)\.[jt]s/,
  fullyParallel: true,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:4173',
    trace: 'on-first-retry'
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } }
  ],
  webServer: {
    command: 'npm run build && npm run preview',
    port: 4173,
    reuseExistingServer: !process.env.CI,
    timeout: 120_000
  }
});
```

## 8. SvelteKit 专用：playwright + SvelteKit 适配

如果使用 SvelteKit，简化 webServer：

```js
// playwright.config.js (SvelteKit)
import { defineConfig } from '@playwright/test';

export default defineConfig({
  webServer: {
    command: 'npm run build && npm run preview',
    port: 4173
  }
});
```
