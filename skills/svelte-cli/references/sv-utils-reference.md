# `sv-utils` reference

> `@sveltejs/sv-utils` is currently **experimental**. The API may change.

`@sveltejs/sv-utils` provides parser-aware transforms, low-level parsers, Svelte config helpers, and package-manager helpers for add-on authors. It has no filesystem awareness — `sv` owns I/O.

```sh
npm install -D @sveltejs/sv-utils
```

---

## `transforms`

A collection of parser-aware functions that accept a callback and return a `(content) => content` function, ready to pass to `sv.file()`.

The parser choice is baked into the transform type — you cannot accidentally parse a Vite config as Svelte.

| Transform | Callback receives | Used for |
| --------- | ----------------- | -------- |
| `transforms.script(cb)` | `{ ast, comments, content, js }` | `.js`, `.ts` files |
| `transforms.svelte(cb)` | `{ ast, content, svelte, js }` | `.svelte` components |
| `transforms.svelteScript(opts, cb)` | `{ ast, content, svelte, js }` (`ast.instance` always non-null) | Components needing guaranteed `<script>` |
| `transforms.css(cb)` | `{ ast, content, css }` | `.css` files |
| `transforms.json(cb)` | `{ data, content, json }` | `package.json`, `tsconfig.json`, etc. |
| `transforms.yaml(cb)` | `{ data, content }` | YAML configs |
| `transforms.toml(cb)` | `{ data, content }` | TOML configs |
| `transforms.text(cb)` | `{ content, text }` | `.env`, `.gitignore`, etc. (string in, string out) |

### Aborting

Return `false` from the callback to abort — the original content is preserved unchanged.

```js
sv.file(
  'eslint.config.js',
  transforms.script(({ ast, js }) => {
    const { value: existing } = js.exports.createDefault(ast, { fallback: myConfig });
    if (existing !== myConfig) return false; // don't touch existing config
    // ...continue AST modifications
  })
);
```

### Standalone usage / unit testing

Transforms are curried:

```js
import { transforms } from '@sveltejs/sv-utils';

const transform = transforms.script(({ ast, js }) => {
  js.imports.addDefault(ast, { as: 'foo', from: 'foo' });
});

const result = transform('export default {}');
// result is the transformed source
```

### Composability

For mixed AST + raw edits, call the curried transform manually inside `sv.file`'s content callback:

```js
sv.file(path, (content) => {
  const transform = transforms.script(({ ast, js }) => {
    js.imports.addDefault(ast, { as: 'foo', from: 'bar' });
  });
  content = transform(content);
  content = content.replace('foo', 'baz');
  return content;
});
```

### Reusable transforms

Export curried transforms from your add-on so multiple files can share them:

```js
// transforms.js
import { transforms } from '@sveltejs/sv-utils';

export const addFooImport = transforms.svelte(({ ast, svelte, js }) => {
  svelte.ensureScript(ast, { language: 'ts' });
  js.imports.addDefault(ast.instance.content, { as: 'Foo', from: './Foo.svelte' });
});
```

```js
sv.file('+page.svelte', addFooImport);
sv.file('index.svelte', addFooImport);
```

---

## Language tooling namespaces

Each callback exposes namespaced helpers. The full list lives in the [sv-utils source](https://github.com/sveltejs/cli/tree/main/packages/sv-utils); common helpers:

### `js.*` — JavaScript / TypeScript

| Helper | Purpose |
| ------ | ------- |
| `js.imports.addDefault(ast, { as, from })` | Add `import X from 'y'` |
| `js.imports.addNamed(ast, { name, as, from })` | Add `import { X } from 'y'` |
| `js.exports.createDefault(ast, { fallback })` | Get-or-create `export default` |
| `js.vite.addPlugin(ast, { code })` | Append a `vite.config` plugin call |
| `js.array.append(node, item)` | Append to an array literal |
| `js.functions.createCall({ name, args, useIdentifiers })` | Build a function-call AST node |
| `js.variables.*` | Variable manipulation |
| `js.objects.*` | Object-literal helpers |

### `css.*` — CSS

| Helper | Purpose |
| ------ | ------- |
| `css.addAtRule(ast, { name, params })` | Add an `@import` / `@tailwind` / etc. |
| `css.addRule` | Append a rule |
| `css.declarations` | Manipulate declarations |

### `svelte.*` — Svelte components

| Helper | Purpose |
| ------ | ------- |
| `svelte.ensureScript(ast, { language })` | Ensure a `<script>` block exists |
| `svelte.addFragment(ast, html)` | Append a parsed HTML fragment to the markup |
| `svelte.addSlot` | Add a `<slot>` |

### `json.*` — JSON

| Helper | Purpose |
| ------ | ------- |
| `json.arrayUpsert(data, key, value)` | Insert or replace an array entry |
| `json.packageScriptsUpsert(data, scripts)` | Insert or update `package.json` scripts |

### `html.*` — HTML

| Helper | Purpose |
| ------ | ------- |
| `html.attributes.*` | Get/set attributes |

### `text.*` — flat text files

| Helper | Purpose |
| ------ | ------- |
| `text.upsertLine(content, line)` | Insert or update a line in `.env`-style files |

---

## Parsers (low-level)

Most add-ons should use `transforms.*` (which handles parsing + error handling automatically). For custom needs use `parse`:

```js
import { parse } from '@sveltejs/sv-utils';

const { ast, generateCode } = parse.script(content);
const { ast, generateCode } = parse.svelte(content);
const { ast, generateCode } = parse.css(content);
const { data, generateCode } = parse.json(content);
const { data, generateCode } = parse.yaml(content);
const { data, generateCode } = parse.toml(content);
const { ast, generateCode } = parse.html(content);
```

Each parser returns the parsed AST (or data) and a `generateCode` function to serialise back to source.

---

## Svelte config helpers

SvelteKit config can live in two places:

1. As the `sveltekit()` plugin argument inside `vite.config.{js,ts}` (default for projects created by `sv`).
2. As the default export of a separate `svelte.config.{js,ts}`.

`svelteConfig` lets add-ons read and edit both transparently — no `kit` nesting to worry about.

### `svelteConfig.edit`

```js
import { svelteConfig } from '@sveltejs/sv-utils';

svelteConfig.edit({ sv, cwd }, ({ ast, property, override, js }) => {
  // Svelte-level option (lives on the config object itself):
  js.array.append(
    property('extensions', { fallback: js.array.create() }),
    '.svx'
  );

  // Kit option (nested under `kit` in svelte.config.js, or flattened in vite.config.js):
  js.imports.addDefault(ast, { from: '@sveltejs/adapter-node', as: 'adapter' });
  override({
    adapter: js.functions.createCall({ name: 'adapter', args: [], useIdentifiers: true })
  });
});
```

| Helper | Purpose |
| ------ | ------- |
| `property(name, { fallback })` | Get-or-create an option's value to mutate in place. |
| `override(props, { dropLeadingComments })` | Set or replace options. `dropLeadingComments` clears a now-stale leading comment (e.g. the `adapter-auto` note when switching adapters). |

The edit is routed through `sv.file`, so it shows up in the same diff as every other file edit.

### `svelteConfig.find` / `svelteConfig.read`

Lower-level helpers, both reading through an injected `read(path)` so detection stays static (the config is **parsed, not executed**).

```js
import { svelteConfig } from '@sveltejs/sv-utils';
import { readFile } from 'node:fs/promises';

const read = (path) => readFile(path, 'utf8').catch(() => null);

svelteConfig.find(read);   // { path, kind } | null   (kind: 'vite' | 'svelte')
svelteConfig.read(read);   // { location, config, kit } | null
```

If both a `svelte.config.js` and a `vite.config.js` are present, `svelte.config` wins.

If neither exists, `svelteConfig.edit` creates a `svelte.config.js` for you.

---

## Package manager helpers

### `pnpm.allowBuilds`

Returns a transform for `pnpm-workspace.yaml` that adds packages to pnpm's allow-builds configuration. Use with `sv.file` when the project uses pnpm.

```js
import { pnpm } from '@sveltejs/sv-utils';

if (packageManager === 'pnpm') {
  sv.file(file.findUp('pnpm-workspace.yaml'), pnpm.allowBuilds('esbuild'));
}
```

The helper detects the installed pnpm version via `pnpm --version`:

| pnpm version | Schema written |
| ------------ | -------------- |
| `>= 11` | Unified `allowBuilds: { pkg: true }` map. Legacy `onlyBuiltDependencies` list (if present) is migrated into the map. |
| `< 11`  | Legacy `onlyBuiltDependencies` list. |

---

## Common patterns

### Edit `vite.config.{js,ts}` (where `sv create` keeps SvelteKit config)

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

### Append a `<script>` import + fragment

```js
sv.file(
  layoutPath,
  transforms.svelteScript({ language: 'ts' }, ({ ast, svelte, js }) => {
    js.imports.addDefault(ast.instance.content, { as: 'Foo', from: './Foo.svelte' });
    svelte.addFragment(ast, '<Foo />');
  })
);
```

### Append to `.env` without a parser

```js
sv.file(
  '.env',
  transforms.text(({ content }) => content + '\nDATABASE_URL="file:local.db"')
);
```

### Update `tsconfig.json` in place

```js
sv.file(
  file.typeConfig,
  transforms.json(({ data }) => {
    data.compilerOptions ??= {};
    data.compilerOptions.strict = true;
  })
);
```

### Switch SvelteKit adapter safely

```js
import { svelteConfig } from '@sveltejs/sv-utils';

svelteConfig.edit({ sv, cwd }, ({ ast, js }) => {
  js.imports.addDefault(ast, { from: '@sveltejs/adapter-vercel', as: 'adapter' });
  // ...override the adapter option
});
```

---

## See also

- `../examples/sv-utils.md` — practical, copy-pasteable examples
- `custom-addon-reference.md` — full add-on lifecycle
- `../SKILL.md` — overview and quick fixes