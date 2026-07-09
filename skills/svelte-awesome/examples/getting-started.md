# Svelte 5 Getting Started Examples

## 1. 创建项目（SvelteKit 推荐）

```bash
# 交互式创建
npx sv create myapp
cd myapp
npm install
npm run dev

# 或非交互式
npx sv create myapp --template minimal --types ts --no-add-ons
```

## 2. 创建首个 Svelte 组件

### `src/routes/+page.svelte`

```svelte
<script lang="ts">
  let name = $state('world');
  let count = $state(0);

  function handleClick() {
    count += 1;
  }
</script>

<h1>Hello {name}!</h1>
<p>count: {count}</p>
<button onclick={handleClick}>click me</button>
```

### 计数器组件 `src/lib/Counter.svelte`

```svelte
<!-- src/lib/Counter.svelte -->
<script lang="ts">
  let { initial = 0 }: { initial?: number } = $props();

  let count = $state(initial);
</script>

<button onclick={() => count++}>
  Clicks: {count}
</button>
```

### 使用计数器

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  import Counter from '$lib/Counter.svelte';
</script>

<Counter initial={5} />
```

## 3. 纯 Vite（非 SvelteKit）

```bash
npm create vite@latest myapp -- --template svelte-ts
cd myapp
npm install
npm run dev
```

### 纯 Svelte 项目结构

```
myapp/
├── src/
│   ├── lib/
│   │   ├── Counter.svelte
│   │   └── stores.svelte.js    # 共享 $state
│   ├── App.svelte
│   └── main.ts
├── public/
├── package.json
├── vite.config.ts
└── tsconfig.json
```

## 4. SvelteKit 路由结构

```
src/routes/
├── +layout.svelte      # 根布局（所有页面共享）
├── +layout.ts          # 布局数据加载
├── +page.svelte        # 首页 /
├── about/
│   ├── +page.svelte   # /about
│   └── +page.ts        # about 页面数据加载
└── [slug]/
    └── +page.svelte   # 动态路由 /:slug
```

## 5. API 路由（SvelteKit）

```ts
// src/routes/api/users/+server.ts
import { json } from '@sveltejs/kit';
import type { RequestHandler } from './$types';

export const GET: RequestHandler = async ({ url }) => {
  const page = Number(url.searchParams.get('page') ?? 1);
  const users = await fetchUsers({ page });
  return json(users);
};

export const POST: RequestHandler = async ({ request }) => {
  const data = await request.json();
  const user = await createUser(data);
  return json(user, { status: 201 });
};
```

## 6. 加载数据（+page.ts）

```ts
// src/routes/posts/[id]/+page.ts
import type { PageLoad } from './$types';
import { error } from '@sveltejs/kit';

export const load: PageLoad = async ({ params, fetch }) => {
  const res = await fetch(`/api/posts/${params.id}`);
  if (!res.ok) throw error(404, 'Post not found');
  return { post: await res.json() };
};
```

```svelte
<!-- src/routes/posts/[id]/+page.svelte -->
<script lang="ts">
  let { data } = $props(); // 来自 load() 的数据
</script>

<h1>{data.post.title}</h1>
```

## 7. 常用 npm scripts

```json
{
  "scripts": {
    "dev": "vite dev",
    "build": "vite build",
    "preview": "vite preview",
    "check": "svelte-kit sync && svelte-check --tsconfig ./tsconfig.json"
  }
}
```

## 8. 配置 svelte.config.js

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-auto';

/** @type {import('@sveltejs/kit').Config} */
const config = {
  compilerOptions: {
    // runes: true // 默认启用
  },
  kit: {
    adapter: adapter()
  }
};

export default config;
```

## 9. TypeScript 配置

```json
// tsconfig.json
{
  "extends": "./.svelte-kit/tsconfig.json",
  "compilerOptions": {
    "strict": true,
    "moduleResolution": "bundler"
  }
}
```

## 10. Svelte 5 Snippet 复用示例

```svelte
<!-- src/lib/PageLayout.svelte -->
<script lang="ts">
  let { title, children }: {
    title: string;
    children: import('svelte').Snippet;
  } = $props();
</script>

<main>
  <h1>{title}</h1>
  {@render children()}
</main>
```

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  import PageLayout from '$lib/PageLayout.svelte';
</script>

<PageLayout title="Welcome">
  {#snippet children()}
    <p>This is the page content!</p>
  {/snippet}
</PageLayout>
```

## 11. VS Code 配置

```json
// .vscode/extensions.json
{
  "recommendations": ["svelte.svelte-vscode"]
}
```

## 12. 环境变量

```
# .env
PUBLIC_API_URL=https://api.example.com
```

```ts
// 在 SvelteKit 中读取
// +page.ts
export const load = ({ fetch }) => {
  const url = import.meta.env.PUBLIC_API_URL;
};
```

```svelte
<!-- 在组件中访问 -->
<script>
  const url = import.meta.env.PUBLIC_API_URL;
</script>
```
