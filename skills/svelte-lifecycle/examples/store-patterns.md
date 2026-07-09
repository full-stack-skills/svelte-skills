# Store Patterns: stores vs $state

## 1. 简单状态用 $state

```js
// counter.svelte.js
export const counter = $state({ value: 0 });
```

```svelte
<!-- Counter.svelte -->
<script>
  import { counter } from './counter.svelte.js';
</script>

<button onclick={() => counter.value++}>{counter.value}</button>
```

## 2. 跨模块共享状态：$state 模块

```js
// user.svelte.js
export const user = $state({ name: '', loggedIn: false });

export function login(name) {
  user.name = name;
  user.loggedIn = true;
}

export function logout() {
  user.name = '';
  user.loggedIn = false;
}
```

```svelte
<script>
  import { user, login, logout } from './user.svelte.js';
</script>

{#if user.loggedIn}
  <p>{user.name} <button onclick={logout}>Logout</button></p>
{:else}
  <button onclick={() => login('Ada')}>Login</button>
{/if}
```

## 3. 何时用 store：异步流

```js
// weather.js
import { readable } from 'svelte/store';

export const weather = readable(null, (set) => {
  let cancelled = false;
  async function poll() {
    while (!cancelled) {
      try {
        const r = await fetch('/api/weather');
        set(await r.json());
      } catch {}
      await new Promise(r => setTimeout(r, 60000));
    }
  }
  poll();
  return () => { cancelled = true; };
});
```

## 4. 何时用 store：计时器驱动

```js
// ticker.js
import { readable } from 'svelte/store';
export const ticker = readable(0, (set) => {
  const id = setInterval(() => set(performance.now()), 16);
  return () => clearInterval(id);
});
```

> `$state` 也能做，但用 readable 让 start/stop 与订阅数挂钩，更优雅。

## 5. 何时用 store：RxJS 互操作

```js
import { readable } from 'svelte/store';
import { fromEvent } from 'rxjs';

export const clicks = readable(0, (set) => {
  const sub = fromEvent(document, 'click').subscribe(() => set($clicks + 1));
  return () => sub.unsubscribe();
});
```

## 6. 持久化 store：localStorage

```js
import { writable } from 'svelte/store';

function persistent(key, initial) {
  const v = typeof localStorage !== 'undefined'
    ? JSON.parse(localStorage.getItem(key) ?? 'null')
    : null;
  const store = writable(v ?? initial);
  store.subscribe((x) => localStorage.setItem(key, JSON.stringify(x)));
  return store;
}

export const todos = persistent('todos', []);
```

## 7. 持久化 store：SSR 安全

```js
import { writable } from 'svelte/store';

function persistent(key, initial) {
  const isBrowser = typeof window !== 'undefined';
  const stored = isBrowser ? localStorage.getItem(key) : null;
  const store = writable(stored ? JSON.parse(stored) : initial);

  if (isBrowser) {
    store.subscribe((v) => localStorage.setItem(key, JSON.stringify(v)));
  }
  return store;
}
```

## 8. sessionStorage 版本

```js
import { writable } from 'svelte/store';

export const session = (() => {
  const key = 'session';
  const v = typeof sessionStorage !== 'undefined'
    ? sessionStorage.getItem(key) : null;
  const store = writable(v ? JSON.parse(v) : null);
  if (typeof sessionStorage !== 'undefined') {
    store.subscribe(x => sessionStorage.setItem(key, JSON.stringify(x)));
  }
  return store;
})();
```

## 9. 跨组件 store：模块单例

```js
// theme.js
import { writable } from 'svelte/store';
export const theme = writable('light');
```

```svelte
<!-- ThemeToggle.svelte -->
<script>
  import { theme } from './theme.js';
</script>

<button onclick={() => $theme = $theme === 'light' ? 'dark' : 'light'}>
  Toggle: {$theme}
</button>
```

```svelte
<!-- App.svelte -->
<script>
  import { theme } from './theme.js';
</script>

<div class:dark={$theme === 'dark'}>...</div>
```

## 10. 派生 store：从 $state 模块

```js
// cart.svelte.js
import { derived } from 'svelte/store';
import { writable } from 'svelte/store';

const items = writable([]);

export const cart = {
  subscribe: items.subscribe,
  add: (item) => items.update(arr => [...arr, item]),
  remove: (id) => items.update(arr => arr.filter(i => i.id !== id))
};

export const total = derived(items, ($items) =>
  $items.reduce((sum, i) => sum + i.price, 0)
);

export const count = derived(items, ($items) => $items.length);
```

## 11. store + $effect 桥接

```svelte
<script>
  import { onDestroy } from 'svelte';
  import { clock } from './clock.js';

  // 在 $effect 中订阅 store
  $effect(() => {
    const unsub = clock.subscribe((v) => {
      // ...
    });
    return unsub;
  });
</script>
```

> 通常 `$clock` 更简单；仅在 effect 体中需要更细控制时这样做。

## 12. 转换 store 为 promise

```js
import { get } from 'svelte/store';
import { user } from './user.js';

export async function requireUser() {
  const u = get(user);
  if (!u) throw new Error('not authenticated');
  return u;
}
```

## 13. 错误处理 store

```js
import { writable } from 'svelte/store';

function createResource() {
  const { subscribe, set, update } = writable({ data: null, error: null, loading: false });

  async function load(fetcher) {
    update(s => ({ ...s, loading: true, error: null }));
    try {
      const data = await fetcher();
      set({ data, error: null, loading: false });
    } catch (error) {
      update(s => ({ ...s, error, loading: false }));
    }
  }

  return { subscribe, load };
}

export const posts = createResource();
```

```svelte
<script>
  import { posts } from './posts.js';
  import { onMount } from 'svelte';

  onMount(() => posts.load(() => fetch('/api/posts').then(r => r.json())));
</script>

{#if $posts.loading}Loading…
{:else if $posts.error}Error: {$posts.error.message}
{:else}{#each $posts.data as p}<p>{p.title}</p>{/each}{/if}
```

## 14. 比对：$state + .svelte.js 是默认

```js
// state.svelte.js — 99% 情况下用这个
export const ui = $state({ sidebarOpen: false, modal: null });
```

```js
// store.js — 仅在需要外部订阅/异步流时
import { readable } from 'svelte/store';
export const connection = readable('online', (set) => { ... });
```

## 15. SSR 安全：context 替代模块 store

```js
// ❌ 模块 store 在 SSR 间共享
// user.js
export const user = writable(null);
```

```svelte
<!-- ❌ 用户 A 的数据泄漏给用户 B -->
<script>
  import { user } from './user.js';
  $user = data.user;
</script>
```

正确做法见 `context-advanced.md`。
