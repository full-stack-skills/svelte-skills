# Building a custom `sv` add-on

> Community add-ons are **experimental**. The API may change. Don't use them in production yet.

This reference covers authoring, testing, bundling, and publishing.

---

## Quick start

Bootstrap from the official template:

```sh
npx sv create --template addon my-addon
cd my-addon
```

The generated project includes `README.md` and `CONTRIBUTING.md` that guide you through the rest.

## Project anatomy

```js
// index.js (the add-on entry point)
import { transforms } from '@sveltejs/sv-utils';
import { defineAddon, defineAddonOptions } from 'sv';

export default defineAddon({
  id: 'my-addon',
  shortDescription: 'Does something useful.',
  options: defineAddonOptions()
    .add('who', { question: 'To whom?', type: 'string' })
    .build(),

  setup: ({ dependsOn, isKit, unsupported }) => {
    if (!isKit) unsupported('Requires SvelteKit');
    dependsOn('vitest');
  },

  run: ({ sv, options, directory }) => {
    sv.file(
      directory.kitRoutes + '/+page.svelte',
      transforms.svelte(({ ast, svelte }) => {
        svelte.addFragment(ast, `<p>Hello ${options.who}!</p>`);
      })
    );
  },

  nextSteps: ({ options }) => [
    `Run \`npm run dev\` to try it out`,
    `Open src/routes/+page.svelte to see the change`
  ]
});
```

---

## Two-package architecture

| Package | Owns |
| ------- | ---- |
| `sv` | **Where and when** — workspace detection, file I/O, dependency tracking, add-on orchestration. |
| `@sveltejs/sv-utils` | **What** — pure parsers, transforms, language tooling. No filesystem awareness. |

This separation means transforms are testable without a workspace and composable across add-ons.

---

## `defineAddon`

```js
import { defineAddon } from 'sv';

defineAddon({
  id,                  // string — used in 'sv add my-addon' (without scope)
  shortDescription,    // string — one-line summary
  homepage,            // optional URL
  options,             // built via defineAddonOptions().build()
  setup,               // ({ dependsOn, unsupported, isKit, kit }) => void
  run,                 // ({ sv, options, cancel, file, language, directory, ... }) => void
  nextSteps,           // ({ options }) => string[]
});
```

### `setup` callback

Called before `run`. Use it to declare environment requirements.

| Helper | Purpose |
| ------ | ------- |
| `dependsOn('vitest')` | Run another add-on first. |
| `unsupported('reason')` | Mark this add-on as incompatible with the current project. |
| `isKit` | `true` if SvelteKit is detected. |

### `run` callback

| Member | Purpose |
| ------ | ------- |
| `sv.file(path, transform)` | Create or edit a file. `transform` can be a string, a `(content) => content` function, or a `transforms.*` curried function. |
| `sv.dependency(name, version)` | Add a runtime dependency. |
| `sv.devDependency(name, version)` | Add a dev dependency. |
| `sv.execute(program, args)` | Run a shell command. |
| `cancel('reason')` | Abort the add-on mid-run. |
| `options` | The values collected from the user. |
| `directory` | Project layout helpers (`kitRoutes`, etc.). |
| `file` | Common file path helpers (`viteConfig`, `stylesheet`, `typeConfig`, etc.). |

---

## `defineAddonOptions`

```js
import { defineAddonOptions } from 'sv';

const options = defineAddonOptions()
  .add('database', {
    question: 'Which database?',
    type: 'select',
    default: 'postgresql',
    options: [
      { value: 'postgresql' },
      { value: 'mysql' },
      { value: 'sqlite' }
    ]
  })
  .add('docker', {
    question: 'Add a docker-compose file?',
    type: 'boolean',
    default: false,
    condition: (opts) => opts.database !== 'sqlite'
  })
  .build();
```

Available field types: `string`, `number`, `boolean`, `select`, `multiselect`.

`condition(opts)` receives the answers collected so far — return `false` to skip the question (its value will be `undefined`). Options are asked in `.add()` order.

---

## Development loop

```sh
# In the add-on package
npm run build     # tsdown bundles to dist/

# In a target project
npx sv add file:../my-addon
```

Iterate: edit -> `npm run build` -> re-run `sv add`. The `demo-add` script in the template rebuilds automatically.

`file:` also works for internal / private add-ons that won't be published — useful for standardising setup across a team.

---

## Testing

`sv/testing` provides `createSetupTest` — a Vitest/Playwright-Test factory that spins up real SvelteKit projects from templates, runs your add-on, and exposes the resulting files.

```js
// tests/addon.test.js
import { expect } from '@playwright/test';
import fs from 'node:fs';
import path from 'node:path';
import { createSetupTest } from 'sv/testing';
import * as vitest from 'vitest';
import addon from '../index.js';

const { test, testCases } = createSetupTest(vitest)(
  { addon },
  {
    kinds: [
      {
        type: 'default',
        options: { 'my-addon': { who: 'World' } }
      }
    ],
    filter: (testCase) => testCase.variant.includes('kit'),
    browser: false
  }
);

test.concurrent.for(testCases)('my-addon $kind.type $variant', async (testCase, ctx) => {
  const cwd = ctx.cwd(testCase);
  const page = fs.readFileSync(path.resolve(cwd, 'src/routes/+page.svelte'), 'utf8');
  expect(page).toContain('Hello World!');
});
```

`vitest.config.js`:

```js
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['tests/**/*.test.{js,ts}'],
    globalSetup: ['tests/setup/global.js']
  }
});
```

`tests/setup/global.js`:

```js
import { fileURLToPath } from 'node:url';
import { setupGlobal } from 'sv/testing';

const TEST_DIR = fileURLToPath(new URL('../../.test-output/', import.meta.url));

export default setupGlobal({ TEST_DIR });
```

---

## Publishing

### Bundling

Add-ons are bundled with [tsdown](https://tsdown.dev/) into a single file. Everything is bundled except `sv` (which is a peer dep).

### `package.json`

```jsonc
{
  "name": "@my-org/sv",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": { "default": "./dist/index.mjs" }
  },
  "publishConfig": { "access": "public" },
  "dependencies": {},                     // must be empty
  "peerDependencies": {
    "sv": "^0.13.0"                       // minimum sv version
  },
  "keywords": ["sv-add", "svelte", "sveltekit"]
}
```

Rules:
- No `dependencies` — bundle everything else.
- `sv` is a peer dependency.
- Add the `sv-add` keyword so users can find your add-on.

### Naming

Must be published under an **npm org**:

```sh
# GOOD
npx sv add @my-org/sv
npx sv add @my-org/core

# BAD — plain names are rejected
npx sv add my-lib
```

If your package is the `sv` scope on the org, it can be omitted:

```sh
npx sv add @my-org          # → @my-org/sv
npx sv add @my-org/sv       # → @my-org/sv
npx sv add @my-org/sv@latest
npx sv add @my-org/sv@1.2.3
```

### Entry points

The CLI looks for `./sv` first, then `.`:

| Strategy | `package.json` exports |
| -------- | ---------------------- |
| Dedicated add-on package | `"exports": { ".": "./dist/addon.mjs" }` |
| Package with other functionality | `"exports": { ".": "./dist/main.mjs", "./sv": "./dist/addon.mjs" }` |

### Publish

```sh
npm login
npm publish
```

`prepublishOnly` runs the build automatically.

---

## Version compatibility

Set a minimum `sv` version in `peerDependencies`. Users get a compatibility warning if their installed `sv` major differs.

```jsonc
"peerDependencies": {
  "sv": "^0.13.0"
}
```

Use the same major as the official add-ons in the [`sveltejs/cli`](https://github.com/sveltejs/cli/tree/main/packages/sv/src/addons) repo to keep behaviour consistent.

---

## Examples

For real-world patterns, read the source of the official add-ons:

- https://github.com/sveltejs/cli/tree/main/packages/sv/src/addons

See `../examples/sv-utils.md` for transform patterns you can copy into your own add-on.