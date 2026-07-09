# Runes Overview Reference

## Runes vs Legacy 对照速查

| 概念 | Svelte 4 (Legacy) | Svelte 5 (Runes) |
|------|-------------------|-------------------|
| 可变状态 | `let x = 0` | `let x = $state(0)` |
| 派生值 | `$: x = a + b` | `let x = $derived(a + b)` |
| 副作用 | `$: { ... }` | `$effect(() => { ... })` |
| Props 声明 | `export let prop` | `let { prop } = $props()` |
| Props 绑定 | `bind:prop` | `bind:prop`（相同）|
| 子组件可变 prop | — | `let { x = $bindable() } = $props()` |

## Runes 生命周期图

```
组件创建
  ↓
$props() 解构        ← 父组件传入
  ↓
$state() 初始化      ← 组件本地状态
  ↓
$derived() 计算      ← 自动从 $state 派生
  ↓
组件渲染            ← DOM 更新
  ↓
$effect() 副作用   ← DOM 操作/第三方库
  ↓
用户交互触发状态变化
  ↓
$derived 重新计算    ← 按需
$effect 重新运行    ← 按需
```

## 模块级 $state（.svelte.js）

```js
// state.svelte.js — 模块级响应式（全局共享）
export const counter = $state({ count: 0 });

// ❌ 不要直接导出 let 声明的 $state
export let count = $state(0); // 编译错误

// ✅ 导出 const 包装的对象
export const countState = $state({ value: 0 });
```

## $state 代理行为详解

```svelte
<script>
  let obj = $state({ a: { b: 1 } });
  let arr = $state([1, 2, { x: 3 }]);

  // ✅ 深层追踪
  obj.a.b = 2;           // 触发更新
  arr[2].x = 4;          // 触发更新
  arr.push({ y: 5 });    // 触发更新
  arr.splice(0, 1);       // 触发更新

  // ❌ 解构后丢失追踪
  let { a, b } = obj;    // a, b 是普通值
  a.b = 99;              // 不触发更新

  // ✅ 正确做法：访问原对象属性
  let copy = obj;         // copy 仍是代理引用
  copy.a.b = 99;         // 触发更新
</script>
```

## $derived 依赖追踪规则

```svelte
<script>
  let a = $state(1);
  let b = $derived(a * 2);  // 依赖 a

  let obj = $state({ x: 10 });
  let val = $derived(obj.x);   // 依赖 obj.x

  // ✅ 解构后仍是响应式
  let { x } = $derived({ a: 1, b: 2 });
  // x 是 $derived(1)，x 变化会触发更新
</script>
```

## $effect 依赖追踪规则

```svelte
<script>
  let count = $state(0);

  // ✅ 追踪 count
  $effect(() => {
    console.log(count);
  });

  // ❌ 不追踪（异步）
  $effect(() => {
    setTimeout(() => console.log(count), 1000);
    // count 变化不会重新运行
  });

  // ❌ 不追踪（已超出追踪上下文）
  $effect(() => {
    someCallback(() => count); // 闭包中读取不追踪
  });
</script>
```

## 跨模块状态传递边界

```
A.svelte.js          B.svelte.js           C.svelte.js
  |                      |                      |
export const state = $state({...})
  |                      |______________________|
  |_____________________________________________|
       ↓
   所有文件引用
   但 compiler 只处理单文件
   所以导出用 const 对象
   而非 let
```
