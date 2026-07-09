# `sv-utils` examples

`@sveltejs/sv-utils` is **experimental**. The API may change. Install as a dev dep in your add-on:

```sh
npm i -D @sveltejs/sv-utils
```

`sv-utils` is pure — no filesystem awareness. `sv` itself owns workspace detection and I/O. This separation lets you test transforms in isolation.

---

## 1. Add a Svelte fragment to `+page.svelte`

```js
import { transforms } from '@sveltejs/sv-utils';

sv.file(
  directory.kitRoutes + '/+page.svelte',
  transforms.svelte(({ ast, svelte }) => {
    svelte.addFragment(ast, '<p>Hello from an add-on!</p>');
  })
);
```

## 2. Edit a Vite config (JS/TS AST)

```js
import { transforms } from '@sveltejs/sv-utils';

sv.file(
  file.viteConfig,
  transforms.script(({ ast, js }) => {
    js.imports.addDefault(ast, { as: 'foo', from: 'foo' });
    js.vite.addPlugin(ast, { code: 'foo()' });
  })
);
```

## 3. Append an `@import` to CSS

```js
import { transforms } from '@sveltejs/sv-utils';

sv.file(
  file.stylesheet,
  transforms.css(({ ast, css }) => {
    css.addAtRule(ast, { name: 'import', params: "'tailwindcss'" });
  })
);
```

## 4. Mutate JSON in place

```js
import { transforms } from '@sveltejs/sv-utils';

sv.file(
  file.typeConfig,
  transforms.json(({ data }) => {
    data.compilerOptions ??= {};
    data.compilerOptions.strict = true;
  })
);
```

The `data` object is mutated; the return value is ignored.

## 5. Append a line to `.env`

```js
import { transforms } from '@sveltejs/sv-utils';

sv.file(
  '.env',
  transforms.text(({ content }) => {
    return content + '\nDATABASE_URL="file:local.db"';
  })
);
```

## 6. Ensure a `<script>` block and add an import + fragment

```js
import { transforms } from '@sveltejs/sv-utils';

sv.file(
  layoutPath,
  transforms.svelteScript({ language: 'ts' }, ({ ast, svelte, js }) => {
    js.imports.addDefault(ast.instance.content, { as: 'Foo', from: './Foo.svelte' });
    svelte.addFragment(ast, '<Foo />');
  })
);
```

`transforms.svelteScript` guarantees `ast.instance` is non-null.

## 7. Abort a transform when content already exists

```js
import { transforms } from '@sveltejs/sv-utils';

sv.file(
  'eslint.config.js',
  transforms.script(({ ast, js }) => {
    const { value: existing } = js.exports.createDefault(ast, { fallback: myConfig });
    if (existing !== myConfig) {
      // config already exists — leave it alone
      return false;
    }
    // ...continue AST modifications
  })
);
```

Returning `false` aborts; the original file content is preserved.

## 8. Edit svelte.config / vite.config sveltekit() call

```js
import { svelteConfig } from '@sveltejs/sv-utils';

svelteConfig.edit({ sv, cwd }, ({ ast, property, override, js }) => {
  // Svelte-level option — append to an array property
  js.array.append(
    property('extensions', { fallback: js.array.create() }),
    '.svx'
  );

  // Kit option — routed automatically
  js.imports.addDefault(ast, { from: '@sveltejs/adapter-node', as: 'adapter' });
  override({
    adapter: js.functions.createCall({ name: 'adapter', args: [], useIdentifiers: true })
  });
});
```

## 9. Locate the Svelte / SvelteKit config statically

```js
import { svelteConfig } from '@sveltejs/sv-utils';
import { readFile } from 'node:fs/promises';

const read = (path) => readFile(path, 'utf8').catch(() => null);

const location = svelteConfig.find(read);    // { path, kind } | null
// kind: 'vite' | 'svelte'
const full    = svelteConfig.read(read);      // { location, config, kit } | null
```

Static detection — the config file is **parsed, not executed**.

## 10. Add a package to pnpm's allow-builds

```js
import { pnpm } from '@sveltejs/sv-utils';

if (packageManager === 'pnpm') {
  sv.file(file.findUp('pnpm-workspace.yaml'), pnpm.allowBuilds('esbuild'));
}
```

Auto-detects pnpm version:
- `>= 11` -> unified `allowBuilds: { pkg: true }` map
- `< 11`  -> legacy `onlyBuiltDependencies` list

---

## Reusable transforms (export from your add-on)

```js
// my-addon/transforms.js
import { transforms } from '@sveltejs/sv-utils';

export const addFooImport = transforms.svelte(({ ast, svelte, js }) => {
  svelte.ensureScript(ast, { language: 'ts' });
  js.imports.addDefault(ast.instance.content, { as: 'Foo', from: './Foo.svelte' });
});
```

```js
// usage
sv.file('+page.svelte', addFooImport);
sv.file('index.svelte', addFooImport);
```

## Composability: mix transforms + raw string edits

```js
import { transforms } from '@sveltejs/sv-utils';

sv.file(path, (content) => {
  const transform = transforms.script(({ ast, js }) => {
    js.imports.addDefault(ast, { as: 'foo', from: 'bar' });
  });
  content = transform(content);
  content = content.replace('foo', 'baz');
  return content;
});
```

Use this when an AST transform doesn't cover what you need.