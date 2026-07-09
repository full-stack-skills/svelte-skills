# $host 完整参考

## 概述

`$host()` 仅在**编译为自定义元素**时可用 —— 返回宿主元素（custom element 根）。

```svelte
<svelte:options customElement="my-stepper" />

<script>
  function dispatch(type) {
    $host().dispatchEvent(new CustomEvent(type));
  }
</script>

<button onclick={() => dispatch('decrement')}>-</button>
<button onclick={() => dispatch('increment')}>+</button>
```

```svelte
<!-- App.svelte -->
<script>
  import './Stepper.svelte';
  let count = $state(0);
</script>

<my-stepper
  ondecrement={() => (count -= 1)}
  onincrement={() => (count += 1)}
></my-stepper>

<p>count: {count}</p>
```

> 文档："When compiling a component as a custom element, the `$host` rune provides access to the host element, allowing you to (for example) dispatch custom events"

## 何时使用

| 场景 | 说明 |
|------|------|
| 派发自定义事件 | `$host().dispatchEvent(new CustomEvent('x', { detail }))` |
| 访问宿主属性 | `$host().getAttribute('foo')` |
| 调用宿主方法 | `$host().focus()` 等 |
| 监听宿主演变 | 通过 `MutationObserver`（一般不必要） |

## 与 Svelte 4 `customElement` 的差异

Svelte 5 中自定义元素编译使用 `<svelte:options customElement="tag-name" />` + `$host()`。Svelte 4 的 `let element` / `onMount` 模式不再需要。

## 典型用例

### 1. 派发事件

```svelte
<svelte:options customElement="my-counter" />
<script>
  let count = $state(0);

  function dispatch() {
    $host().dispatchEvent(
      new CustomEvent('change', {
        detail: { count },
        bubbles: true,
        composed: true
      })
    );
  }
</script>

<button onclick={() => { count += 1; dispatch(); }}>
  count: {count}
</button>
```

### 2. 访问宿主的 ARIA / role

```svelte
<svelte:options customElement="my-widget" />
<script>
  $effect(() => {
    const el = $host();
    el.setAttribute('aria-label', 'my widget');
  });
</script>
```

### 3. 响应式属性变化（继承 attributes）

```svelte
<svelte:options customElement="my-display" customElementObservers={{ attr: ['data-foo'] }} />
<script>
  let value = $state(0);
  // $host 上的 attribute 变化会触发重新渲染
</script>
```

> Svelte 自动观察自定义元素 attribute 并传给组件。`customElementObservers` 配置哪些 attribute 变化触发更新。

### 4. 透传 slot 内容

```svelte
<svelte:options customElement="my-card" />
<script>
  // slots 在 custom element 中通过 children 处理
</script>

<div class="card">
  <slot />
</div>
```

### 5. 暴露 imperative API

```svelte
<svelte:options customElement="my-stepper" />
<script>
  let count = $state(0);

  // 暴露方法（通过 export）
  export function reset() {
    count = 0;
  }
</script>

<button onclick={() => (count += 1)}>{count}</button>
```

外部调用：`document.querySelector('my-stepper').reset()`。

## $host vs 普通 DOM 访问

```svelte
<script>
  // ❌ 在 Svelte 组件中用 document.querySelector 找自己
  // const self = document.querySelector('my-stepper');

  // ✅ 用 $host
  const self = $host();
</script>
```

> `$host()` 只在 custom element 模式下有效，普通组件调用会编译错误。

## 错误用法

### ❌ 在普通组件中调用

```svelte
<!-- 普通 .svelte 文件，非 custom element -->
<script>
  $host(); // ❌ 编译错误
</script>
```

### ❌ 用 $host 替代 querySelector

```svelte
<!-- custom element 内部 -->
<script>
  function findChild() {
    // ❌ 不要用 $host().querySelector 找内部元素
    // 内部结构改变会破坏代码
    // 改用 bind:this
    return $host().querySelector('.child');
  }
</script>
```

## 自定义元素编译配置

`<svelte:options>` 接受：

```svelte
<svelte:options
  customElement="my-tag"
  customElementObservers={{ attr: ['data-foo'], prop: ['value'] }}
  customElementEvents={['change', 'input'] }}
  customElementShadow="open"  // 或 "closed" / "none"
/>
```

## 完整示例：可复用计数器组件

```svelte
<!-- Counter.svelte -->
<svelte:options
  customElement="my-counter"
  customElementEvents={['count']}
  customElementShadow="open"
/>

<script>
  let { initial = 0 } = $props();
  let count = $state(initial);

  function emit() {
    $host().dispatchEvent(
      new CustomEvent('count', {
        detail: { count },
        bubbles: true,
        composed: true
      })
    );
  }
</script>

<button onclick={() => { count -= 1; emit(); }}>-</button>
<span>{count}</span>
<button onclick={() => { count += 1; emit(); }}>+</button>
```

```html
<!-- index.html -->
<my-counter initial="5" oncount="(e) => console.log(e.detail)"></my-counter>
```

## SSR 行为

- 自定义元素是浏览器 API
- SSR 时 `$host()` 不调用（custom element 不在 Node 中渲染）
- 若要在 SSR 期间用此组件，框架会跳过 `<my-counter>` 或在客户端重建

## 性能 & 注意

1. **Shadow DOM 隔离**：默认 open Shadow DOM，外部样式不进入
2. **事件冒泡**：跨越 Shadow DOM 边界需要 `composed: true`
3. **属性同步**：HTML attribute → Svelte prop 的同步在 `customElementObservers` 中配置
4. **避免重名 tag**：HTML tag 必须包含连字符（HTML 自定义元素规范）
