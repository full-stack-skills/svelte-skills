# Template Syntax Reference

## {#if} / {:else if} / {:else}

```svelte
{#if user}
  <p>Welcome {user.name}</p>
{:else if pending}
  <p>Loading...</p>
{:else}
  <p>Please log in</p>
{/if}
```

## {#each} 完整用法

```svelte
{#each items as item (item.id)}
  <li>
    {item.name} × {item.qty}
  </li>
{:else}
  <li>列表为空</li>
{/each}

<!-- 解构 -->
{#each items as { id, name, ...rest } (id)}
  <li>{name} <Extra {...rest} /></li>
{/each}

<!-- 嵌套 key -->
{#each matrix as row (row.id)}
  {#each row.cells as cell (cell.id)}
    <td>{cell.value}</td>
  {/each}
{/each}
```

## {#await} 完整用法

```svelte
{#await fetchData()}
  <p>加载中...</p>
{:then data}
  <p>数据: {data.value}</p>
{:catch err}
  <p>错误: {err.message}</p>
{/await}

<!-- 仅成功路径 -->
{#await promise then result}
  <p>{result}</p>
{/await}

<!-- 仅错误路径 -->
{#await promise catch err}
  <p>{err}</p>
{/await}

<!-- 懒加载组件 -->
{#await import('./Heavy.svelte') then { default: Component }}
  <Component />
{/await}
```

## Snippet 完整用法

```svelte
<!-- 定义 -->
{#snippet footer(items)}
  <ul>
    {#each items as item}
      <li>{item}</li>
    {/each}
  </ul>
{/snippet}

<!-- 调用 -->
{@render footer(['a', 'b', 'c'])}

<!-- 条件 snippet -->
{#if showHeader}
  {#snippet header()}
    <h1>Title</h1>
  {/snippet}
{/if}
{@render header?.()}
```

## @render / @html / @debug

```svelte
<!-- @render: 渲染 snippet -->
{@render snippetName(arg)}

<!-- @html: 原始 HTML（XSS 风险）-->
{@html '<strong>粗体</strong>'}

<!-- @debug: 调试输出 -->
{@debug user, count}           <!-- 指定变量 -->
{@debug}                       <!-- 任意变化 -->

<!-- @const: 局部常量（Svelte 5 推荐用 let/const 声明标签） -->
{#each items as item}
  {@const price = item.price * item.qty}
  <p>{item.name}: {price}</p>
{/each}
```

## bind: 完整对照

| 绑定目标 | 示例 | 方向 |
|---------|------|------|
| value | `<input bind:value />` | 双向 |
| checked | `<input type="checkbox" bind:checked />` | 双向 |
| group | `<input type="radio" bind:group />` | 双向 |
| files | `<input type="file" bind:files />` | 双向 |
| clientWidth | `<div bind:clientWidth />` | 单向（readonly）|
| clientHeight | `<div bind:clientHeight />` | 单向（readonly）|
| this | `<canvas bind:this />` | 双向 |
