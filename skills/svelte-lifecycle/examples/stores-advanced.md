# Stores Advanced Examples

## 1. writable 基础

```js
import { writable } from 'svelte/store';

const count = writable(0);
count.set(1);
count.update(n => n + 1);
```

## 2. writable 带 start/stop

第一个订阅者订阅时调用 start，返回的函数在最后一个订阅者退订时调用：

```js
import { writable } from 'svelte/store';

const conn = writable(null, () => {
  const ws = new WebSocket('wss://example.com');
  ws.addEventListener('message', (e) => conn.set(JSON.parse(e.data)));
  return () => ws.close();
});
```

## 3. readable 时间流

```js
import { readable } from 'svelte/store';

export const clock = readable(new Date(), (set) => {
  set(new Date());
  const id = setInterval(() => set(new Date()), 1000);
  return () => clearInterval(id);
});
```

```svelte
<script>
  import { clock } from './clock.js';
</script>

<p>Current: {$clock.toLocaleTimeString()}</p>
```

## 4. readable 交替值（使用 update）

```js
import { readable } from 'svelte/store';

export const ticktock = readable('tick', (set, update) => {
  const id = setInterval(() => {
    update(sound => sound === 'tick' ? 'tock' : 'tick');
  }, 1000);
  return () => clearInterval(id);
});
```

## 5. derived 基础

```js
import { derived, writable } from 'svelte/store';

const a = writable(2);
const b = writable(3);
export const sum = derived([a, b], ([$a, $b]) => $a + $b);
```

## 6. derived 异步 + 初始值

```js
import { derived, writable } from 'svelte/store';

const query = writable('react');
export const results = derived(
  query,
  ($q, set) => {
    const ac = new AbortController();
    fetch(`/api/search?q=${$q}`, { signal: ac.signal })
      .then(r => r.json())
      .then(set)
      .catch(() => {});
    return () => ac.abort();
  },
  []                                  // 初始值
);
```

## 7. derived 维护定时器

```js
import { derived, writable } from 'svelte/store';

const frequency = writable(1);
export const ticks = derived(
  frequency,
  ($f, set) => {
    const id = setInterval(() => set(Date.now()), 1000 / $f);
    return () => clearInterval(id);
  },
  Date.now()
);
```

> 关键：返回的清理函数在 frequency 变化时**先**被调用，**再**重启新 interval。

## 8. 自定义 store（满足 store 契约）

```js
function createGeolocation() {
  let current = { lat: 0, lng: 0 };
  const subscribers = new Set();

  const subscribe = (fn) => {
    subscribers.add(fn);
    fn(current);
    const id = navigator.geolocation.watchPosition(
      (pos) => {
        current = { lat: pos.coords.latitude, lng: pos.coords.longitude };
        subscribers.forEach(fn => fn(current));
      },
      () => {}
    );
    return () => {
      subscribers.delete(fn);
      navigator.geolocation.clearWatch(id);
    };
  };

  return { subscribe };
}

export const geo = createGeolocation();
```

## 9. readonly 包装只读视图

```js
import { readonly, writable } from 'svelte/store';

const _settings = writable({ theme: 'light' });
export const settings = readonly(_settings);  // 外部不能 .set
```

```svelte
<script>
  import { _settings } from './settings.js';
  function toggle() { _settings.update(s => ({ ...s, theme: 'dark' })); }
</script>
```

## 10. get 同步读一次

```js
import { get, writable } from 'svelte/store';

const count = writable(5);
console.log(get(count));  // 5
```

> 不建议在热路径用：内部会订阅 → 读 → 退订，开销较大。

## 11. $store 自动订阅（语法糖）

```svelte
<script>
  import { writable } from 'svelte/store';

  const user = writable({ name: 'Ada' });
  // $user 在组件初始化时自动订阅，销毁时自动退订
</script>

<p>Hello {$user.name}</p>
<button onclick={() => $user.name = 'Bob'}>Rename</button>
```

## 12. 跨模块 store 单例

```js
// user.js
import { writable } from 'svelte/store';
export const user = writable(null);
```

```svelte
<!-- Login.svelte -->
<script>
  import { user } from './user.js';
  function login() { $user = { name: 'Ada' }; }
</script>

<!-- Header.svelte -->
<script>
  import { user } from './user.js';
</script>

{#if $user}<p>Hi {$user.name}</p>{/if}
```

## 13. 显式 subscribe 替代 $ 自动订阅

```svelte
<script>
  import { onDestroy } from 'svelte';
  import { clock } from './clock.js';

  let now;
  const unsub = clock.subscribe(v => now = v);
  onDestroy(unsub);
</script>
```

> 大多数情况下 `$clock` 语法更简单。仅在需要脱离模板的纯 JS 上下文（如外部函数）时手动 subscribe。

## 14. store 派生为普通值

```js
import { derived, writable } from 'svelte/store';

const items = writable([1, 2, 3, 4, 5]);
export const evens = derived(items, ($items) => $items.filter(n => n % 2 === 0));
```

## 15. 自定义可写 store

```js
import { writable } from 'svelte/store';

function persistentStore(key, initial) {
  const stored = typeof localStorage !== 'undefined'
    ? localStorage.getItem(key)
    : null;
  const start = stored !== null ? JSON.parse(stored) : initial;
  const store = writable(start);

  store.subscribe((v) => {
    if (typeof localStorage !== 'undefined') {
      localStorage.setItem(key, JSON.stringify(v));
    }
  });

  return store;
}

export const prefs = persistentStore('prefs', { theme: 'light', lang: 'en' });
```
