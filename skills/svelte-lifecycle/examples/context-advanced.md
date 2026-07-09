# Context API Examples

## 1. createContext + TypeScript（推荐，Svelte 5.40+）

```ts
// context.ts
import { createContext } from 'svelte';

interface User { name: string }

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

## 2. setContext + getContext（key-based 备选）

```svelte
<!-- Parent.svelte -->
<script>
  import { setContext } from 'svelte';
  setContext('theme', 'dark');
</script>
```

```svelte
<!-- Child.svelte -->
<script>
  import { getContext } from 'svelte';
  const theme = getContext('theme');
</script>
```

键与值可以是任意 JS 值。`createContext` 更类型安全。

## 3. 任何值都可以作为 context

```svelte
<script>
  import { setContext } from 'svelte';
  import { writable } from 'svelte/store';

  const cart = writable([]);
  setContext('cart', cart);
  setContext('user', { name: 'Ada' });
  setContext('flags', Symbol('dev'));
</script>
```

## 4. 与 $state 组合：计数共享

```ts
// counter.ts
import { createContext } from 'svelte';
interface Counter { count: number }
export const [getCounter, setCounter] = createContext<Counter>();
```

```svelte
<!-- App.svelte -->
<script>
  import { setCounter } from './counter.ts';
  import Child from './Child.svelte';

  const counter = $state({ count: 0 });
  setCounter(counter);
</script>

<button onclick={() => counter.count++}>+</button>
<button onclick={() => counter.count = 0}>reset</button>
<Child />
<Child />
<Child />
```

```svelte
<!-- Child.svelte -->
<script>
  import { getCounter } from './counter.ts';
  const counter = getCounter();
</script>

<p>count = {counter.count}</p>
```

## 5. 错误：重新赋值 $state 对象破坏响应式

```svelte
<!-- ❌ 不响应 -->
<script>
  import { getCounter } from './counter.ts';
  const counter = getCounter();
</script>

<button onclick={() => counter = { count: 0 } }>reset</button>
```

```svelte
<!-- ✅ 原地修改 -->
<script>
  import { getCounter } from './counter.ts';
  const counter = getCounter();
</script>

<button onclick={() => counter.count = 0}>reset</button>
```

Svelte 会发出警告。

## 6. 通过 context 传递函数 getter（用于原始值）

```svelte
<!-- Parent -->
<script>
  import { setContext } from 'svelte';
  let count = $state(0);
  setContext('count', () => count);
</script>
```

```svelte
<!-- Child -->
<script>
  import { getContext } from 'svelte';
  const getCount = getContext('count');
</script>

<p>{getCount()}</p>
```

> context 中的原始值不会自动响应——必须传函数或 `$state` 对象。

## 7. hasContext 判断

```svelte
<script>
  import { hasContext, getContext } from 'svelte';
  if (hasContext('theme')) {
    const theme = getContext('theme');
  } else {
    // fallback 默认值
  }
</script>
```

常用于：组件是 optional 用户，可独立运行也可嵌入有上下文的父组件。

## 8. getAllContexts

```svelte
<script>
  import { getAllContexts } from 'svelte';
  const all = getAllContexts();
  console.log([...all.keys()]);
</script>
```

常用于调试或库的内部实现。

## 9. hasContext + getContext：可选上下文

```svelte
<!-- ThemeAware.svelte -->
<script>
  import { hasContext, getContext } from 'svelte';
  const theme = hasContext('theme')
    ? getContext('theme')
    : 'light';
</script>

<div class={theme}>...</div>
```

## 10. SSR 安全：用 Context 替代模块全局 state

```svelte
<!-- ❌ user.svelte.ts 中导出的 $state 在 SSR 下会在请求间共享 -->
<!-- user.svelte.ts -->
export const user = $state({ name: '' });
```

```svelte
<!-- ✅ 使用 context 保证每个请求独立 -->
<script>
  import { setContext } from 'svelte';
  let { data } = $props();
  setContext('user', $state({ name: data.user.name }));
</script>
```

## 11. Component testing 包装器（Svelte 5.49+）

```js
// MyComponent.test.js
import { mount, unmount } from 'svelte';
import { expect, test } from 'vitest';
import { setUser } from './context';
import MyComponent from './MyComponent.svelte';

test('renders user', () => {
  function Wrapper(...args) {
    setUser({ name: 'Bob' });
    return MyComponent(...args);
  }

  const c = mount(Wrapper, { target: document.body });
  expect(document.body.innerHTML).toBe('<h1>Hello Bob!</h1>');
  unmount(c);
});
```

## 12. i18n context

```ts
// i18n.ts
import { createContext } from 'svelte';

export type Translator = (key: string) => string;

const translations = {
  en: { hello: 'Hello' },
  zh: { hello: '你好' }
};

export const [getI18n, setI18n] = createContext<{
  t: Translator;
  locale: string;
}>();
```

```svelte
<!-- App.svelte -->
<script>
  import { setI18n } from './i18n';
  import Child from './Child.svelte';

  let locale = $state('en');
  const dict = $derived(translations[locale]);
  const t = (key) => dict[key] ?? key;

  setI18n({ t, get locale() { return locale; } });
</script>

<button onclick={() => locale = locale === 'en' ? 'zh' : 'en'}>
  Switch
</button>
<Child />
```

```svelte
<!-- Child.svelte -->
<script>
  import { getI18n } from './i18n';
  const { t } = getI18n();
</script>

<p>{t('hello')}</p>
```

## 13. Theme context 实际使用

```svelte
<!-- App.svelte -->
<script>
  import { setContext } from 'svelte';
  let theme = $state('light');
  setContext('theme', {
    get value() { return theme; },
    set: (v) => theme = v
  });
</script>

<button onclick={() => theme = theme === 'light' ? 'dark' : 'light'}>
  Toggle
</button>
<slot />
```

## 14. createContext 返回 [get, set] 顺序

```ts
import { createContext } from 'svelte';

// 总是 [get, set]
const [getX, setX] = createContext<X>();
```

> 注意：解构顺序永远是 `get` 在前，`set` 在后。

## 15. 在事件处理器外 setContext

```svelte
<!-- ❌ setContext 必须在 setup 阶段（同步、组件初始化时） -->
<script>
  import { setContext } from 'svelte';
  function bad() {
    setContext('key', 'value');  // 错误！事件处理器内调用
  }
</script>
```

```svelte
<!-- ✅ 组件初始化时调用 -->
<script>
  import { setContext } from 'svelte';
  setContext('key', 'value');
  function good() {
    // 这里只读，不调用 setContext
  }
</script>
```
