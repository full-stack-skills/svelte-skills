# {#if} / {#each} / {#await} / {#snippet} Patterns

## 1. {#if} 完整形式

```svelte
{#if user.loggedIn}
  <p>Welcome, {user.name}!</p>
{:else if user.pending}
  <p>Loading...</p>
{:else}
  <p>Please log in.</p>
{/if}
```

## 2. {#if} + {#each} 嵌套

```svelte
{#if users.length > 0}
  <ul>
    {#each users as user (user.id)}
      <li>
        {user.name}
        {#if user.admin}
          <span class="badge">Admin</span>
        {/if}
      </li>
    {/each}
  </ul>
{:else}
  <p>No users found.</p>
{/if}
```

## 3. {#each} 带索引和键

```svelte
{#each items as item, i (item.id)}
  <option value={item.id}>
    {i + 1}. {item.label}
  </option>
{/each}
```

## 4. {#each} 对象/Map 遍历

```svelte
<script>
  let map = $state(new Map([['a', 1], ['b', 2]]));
  let obj = $state({ x: 1, y: 2 });
</script>

{#each [...map] as [key, value]}
  <p>{key}: {value}</p>
{/each}

{#each Object.entries(obj) as [k, v]}
  <p>{k}: {v}</p>
{/each}
```

## 5. {#each} + keyed（唯一键）

```svelte
<!-- ✅ 正确：使用唯一 id -->
{#each todos as todo (todo.id)}
  <TodoItem {todo} />
{/each}

<!-- ❌ 错误：用 index -->
{#each todos as todo, i}
  <TodoItem {todo} />
{/each}
```

## 6. {#await} 三段式

```svelte
{#await promise}
  <p>Loading...</p>
{:then data}
  <p>Result: {data}</p>
{:catch error}
  <p>Error: {error.message}</p>
{/await}
```

## 7. {#await} 只处理成功

```svelte
{#await promise then data}
  <p>{data}</p>
{/await}
```

## 8. {#await} + 变量

```svelte
{#await fetchUser(id) then user}
  <p>{user.name}</p>
{:catch error}
  <p>Failed: {error.message}</p>
{/await}
```

## 9. {#snippet} 定义和调用

```svelte
{#snippet hello(name)}
  <p>Hello, {name}!</p>
{/snippet}

{@render hello('Alice')}
{@render hello('Bob')}
```

## 10. {#snippet} 条件渲染

```svelte
<script>
  let show = $state(true);
</script>

{#snippet card(title)}
  <div class="card">
    <h3>{title}</h3>
    <slot />
  </div>
{/snippet}

{#if show}
  {@render card('My Card')}
{/if}
```

## 11. {#snippet} 递归（树节点）

```svelte
{#snippet tree(nodes)}
  <ul>
    {#each nodes as node (node.id)}
      <li>
        {node.name}
        {#if node.children?.length}
          {@render tree(node.children)}
        {/if}
      </li>
    {/each}
  </ul>
{/snippet}

{@render tree(rootNodes)}
```

## 12. {#snippet} 作为 Props 传递

```svelte
<!-- List.svelte -->
<script>
  let { items, renderItem }: {
    items: any[];
    renderItem: import('svelte').Snippet<[any]>;
  } = $props();
</script>

{#each items as item}
  {@render renderItem(item)}
{/each}

<!-- Parent.svelte -->
<List {items}>
  {#snippet renderItem(item)}
    <span class="item">{item.name}</span>
  {/snippet}
</List>
```

## 13. {#await} + fork（Svelte 5.42+）

```svelte
<script>
  import { fork } from 'svelte';

  let pending = $state(null);

  function preload() {
    pending ??= fork(() => {
      open = true;
    });
  }
</script>
```

## 14. {#each} 解构

```svelte
{#each items as { id, name, details: { age, city } }}
  <p>{id}: {name}, {age} in {city}</p>
{/each}
```
