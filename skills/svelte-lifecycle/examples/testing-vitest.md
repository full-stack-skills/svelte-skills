# Testing with Vitest Examples

## 1. Vitest + runes 单元测试

```js
// multiplier.svelte.js
export function multiplier(initial, k) {
  let count = $state(initial);
  return {
    get value() { return count * k; },
    set: (c) => { count = c; }
  };
}
```

```js
// multiplier.svelte.test.js
import { flushSync } from 'svelte';
import { expect, test } from 'vitest';
import { multiplier } from './multiplier.svelte.js';

test('Multiplier', () => {
  const double = multiplier(0, 2);
  expect(double.value).toBe(0);
  double.set(5);
  expect(double.value).toBe(10);
});
```

> 文件名包含 `.svelte.` 时 Vitest 让 runes 可用。

## 2. 测试文件中使用 $state

```js
// counter.svelte.test.js
import { flushSync } from 'svelte';
import { expect, test } from 'vitest';

test('Counter', () => {
  let count = $state(0);
  expect(count).toBe(0);
  count = 5;
  flushSync();
  expect(count).toBe(5);
});
```

## 3. $effect 在测试中

```js
// logger.svelte.test.js
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

## 4. 组件测试：mount + flushSync

```js
// Counter.test.js
import { mount, unmount, flushSync } from 'svelte';
import { expect, test } from 'vitest';
import Counter from './Counter.svelte';

test('Counter increments', () => {
  const c = mount(Counter, {
    target: document.body,
    props: { initial: 0 }
  });

  expect(document.body.innerHTML).toBe('<button>0</button>');

  document.body.querySelector('button').click();
  flushSync();

  expect(document.body.innerHTML).toBe('<button>1</button>');
  unmount(c);
});
```

## 5. 用 @testing-library/svelte

```js
import { render, screen } from '@testing-library/svelte';
import userEvent from '@testing-library/user-event';
import { expect, test } from 'vitest';
import Counter from './Counter.svelte';

test('Counter via testing-library', async () => {
  const user = userEvent.setup();
  render(Counter);
  const btn = screen.getByRole('button');
  expect(btn).toHaveTextContent('0');
  await user.click(btn);
  expect(btn).toHaveTextContent('1');
});
```

## 6. Context wrapper 测试

```js
// MyComponent.test.js
import { mount, unmount } from 'svelte';
import { expect, test } from 'vitest';
import { setUser } from './context';
import MyComponent from './MyComponent.svelte';

test('renders with user', () => {
  function Wrapper(...args) {
    setUser({ name: 'Bob' });
    return MyComponent(...args);
  }

  const c = mount(Wrapper, { target: document.body });
  expect(document.body.innerHTML).toBe('<h1>Hello Bob!</h1>');
  unmount(c);
});
```

## 7. Store 测试

```js
// counter.store.test.js
import { get } from 'svelte/store';
import { expect, test } from 'vitest';
import { counter } from './counter.js';

test('counter', () => {
  expect(get(counter)).toBe(0);
  counter.set(5);
  expect(get(counter)).toBe(5);
  counter.update(n => n + 1);
  expect(get(counter)).toBe(6);
});
```

## 8. 异步 effect 清理

```js
// async.test.js
import { flushSync } from 'svelte';
import { expect, test, vi } from 'vitest';

test('async effect with abort', () => {
  const cleanup = $effect.root(() => {
    let id = 1;
    const log = [];
    $effect(() => {
      const ac = new AbortController();
      fetch(`/api/${id}`, { signal: ac.signal })
        .then(r => r.json())
        .then(d => log.push(d));
      return () => ac.abort();
    });

    flushSync();
    id = 2;
    flushSync();
    // 第一个 fetch 应被 abort
  });
  cleanup();
});
```

## 9. vite.config.js 完整配置

```js
// vite.config.js
import { defineConfig } from 'vitest/config';
import { svelte } from '@sveltejs/vite-plugin-svelte';

export default defineConfig({
  plugins: [svelte({ hot: false })],
  test: {
    environment: 'jsdom',
    include: ['**/*.test.{js,ts}', '**/*.svelte.test.{js,ts}']
  },
  resolve: process.env.VITEST
    ? { conditions: ['browser'] }
    : undefined
});
```

## 10. 浏览器条件注释

在不需要 `environment: 'jsdom'` 的项目中，可在文件顶部声明：

```js
// @vitest-environment node
import { expect, test } from 'vitest';
```

或针对单文件用 jsdom：

```js
// @vitest-environment jsdom
import { mount } from 'svelte';
```

## 11. Mock fetch

```js
import { vi, expect, test, beforeEach } from 'vitest';

beforeEach(() => {
  globalThis.fetch = vi.fn(() =>
    Promise.resolve({ json: () => Promise.resolve({ name: 'Bob' }) })
  );
});

test('loads user', async () => {
  // ...
});
```

## 12. 测试两向绑定

```svelte
<!-- Input.svelte -->
<script>
  let value = $state('');
</script>

<input bind:value />
<p>{value}</p>
```

```js
// Input.test.js
import { mount, unmount, flushSync } from 'svelte';
import { expect, test } from 'vitest';
import Input from './Input.svelte';

test('two-way binding', () => {
  const c = mount(Input, { target: document.body });
  const input = document.body.querySelector('input');

  input.value = 'hello';
  input.dispatchEvent(new Event('input'));
  flushSync();

  expect(document.body.querySelector('p').textContent).toBe('hello');
  unmount(c);
});
```

## 13. 测试 snippet props

```js
// WrapperComponent.test.js
import { mount, unmount, flushSync } from 'svelte';
import { expect, test } from 'vitest';
import Parent from './Parent.svelte';

test('renders children snippet', () => {
  function Wrapper(...args) {
    return Parent({
      ...args,
      props: {
        ...args.props,
        children: () => '<span>child</span>'
      }
    });
  }

  const c = mount(Wrapper, { target: document.body });
  expect(document.body.innerHTML).toContain('child');
  unmount(c);
});
```
