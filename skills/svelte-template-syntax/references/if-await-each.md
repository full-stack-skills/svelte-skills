# {#if} / {#each} / {#await} 完整参考

## {#if} 语法

```svelte
{#if condition}
  <p>True</p>
{:else if otherCondition}
  <p>Other true</p>
{:else}
  <p>All false</p>
{/if}
```

规则：
- `condition` 可以是任何表达式
- `{:else if}` 和 `{:else}` 可选
- 嵌套层数无限制

## {#each} 基本语法

```svelte
{#each expression as item (key)}
  <div>{item.name}</div>
{:else}
  <p>No items</p>
{/each}
```

### Keyed each blocks

**必须使用唯一键**（不能用 index）：

```svelte
<!-- ✅ 正确 -->
{#each todos as todo (todo.id)}
  <TodoItem {todo} />
{/each}

<!-- ❌ 错误：index 不是稳定键 -->
{#each todos as todo, i}
  <TodoItem {todo} />
{/each}
```

Keyed 机制：Svelte 用 key 追踪每个节点，变化时只更新对应 DOM 节点。

### 解构

```svelte
{#each items as { id, name, meta: { count } }}
  <p>{id}: {name} ({count})</p>
{/each}
```

### 对象/Map 遍历

```svelte
{#each Object.entries(obj) as [key, value]}
  <p>{key}: {value}</p>
{/each}

{#each [...map] as [key, value]}
  <p>{key}: {value}</p>
{/each}
```

## {#await} 语法

```svelte
{#await promise}
  <p>Loading...</p>
{:then data}
  <p>{data}</p>
{:catch error}
  <p>Error: {error.message}</p>
{/await}
```

### 只处理成功

```svelte
{#await promise then data}
  <p>{data}</p>
{/await}
```

### await + $derived

```svelte
let result = $derived(await somePromise());
```

> ⚠️ 需启用 `experimental.async` 配置

## {#key} 强制重建

```svelte
<!-- count 变化时重建整个 div -->
{#key count}
  <div>{Math.random()}</div>
{/key}
```

## {:else} 与 {#each}

`{:else}` 在数组为空时渲染：

```svelte
{#each items as item (item.id)}
  <p>{item.name}</p>
{:else}
  <p>No items found.</p>
{/each}
```
