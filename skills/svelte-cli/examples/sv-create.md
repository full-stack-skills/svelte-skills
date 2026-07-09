# `sv create` examples

`npx sv create [options] [path]` scaffolds a SvelteKit project.

---

## 1. Interactive (default)

```sh
npx sv create my-app
cd my-app
```

Walks through template choice, types, add-ons, and install step.

## 2. Minimal template, TypeScript, no install

```sh
npx sv create --template minimal --types ts --no-install my-app
```

## 3. Demo template (word-guessing game)

```sh
npx sv create --template demo my-app
```

## 4. Library template (for publishing a Svelte component library)

```sh
npx sv create --template library my-lib
```

Sets up `svelte-package` with `exports` field, `pkg` field, and a build pipeline.

## 5. From a Svelte playground URL

```sh
npx sv create --from-playground "https://svelte.dev/playground/hello-world" my-app
```

Downloads playground files, detects external deps, and sets up the SvelteKit project.

## 6. Pre-pick add-ons during creation

```sh
npx sv create --add eslint prettier playwright my-app
```

Same syntax as `sv add` — multiple add-ons are space-separated.

## 7. Skip the add-ons prompt entirely

```sh
npx sv create --no-add-ons --template minimal my-app
```

## 8. JSDoc-typed project (no TS toolchain)

```sh
npx sv create --types jsdoc my-app
```

Uses JSDoc syntax for types — no separate TS build step required.

## 9. Choose a package manager explicitly

```sh
npx sv create --install pnpm my-app
npx sv create --install bun my-app
npx sv create --install yarn my-app
npx sv create --install deno my-app
npx sv create --install npm my-app
```

## 10. Allow scaffolding into a non-empty directory

```sh
npx sv create --no-dir-check my-app
```

Skips the "directory not empty" check. Useful for re-scaffolding into an existing folder.

---

## Common workflows

### Fully-scripted CI / non-interactive create

```sh
npx sv create \
  --template minimal \
  --types ts \
  --no-add-ons \
  --no-install \
  --no-dir-check \
  my-app
cd my-app && npm install
```

### Create with full tooling in one shot

```sh
npx sv create --template minimal --types ts --add eslint prettier vitest playwright --install pnpm my-app
```

### Create + add a community add-on

```sh
npx sv create --add eslint @supacool my-app
```