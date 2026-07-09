# Project Creation Examples

`npx sv create` is the official scaffolder for SvelteKit. The CLI supports interactive and non-interactive flags. All snippets target the latest `sv` CLI.

## 1. Interactive create (default)

```sh
npx sv create my-app
cd my-app
npm install
npm run dev
# http://localhost:5173
```

The CLI prompts for: template, TypeScript, add-ons (Prettier, ESLint, Vitest, Playwright, Tailwind, etc.), and package manager.

## 2. Non-interactive create with all flags

```sh
npx sv create my-app \
  --template minimal \
  --types ts \
  --no-add-ons \
  --install npm
```

## 3. Pick a template

```sh
# Demo template (richer starter)
npx sv create my-app --template demo

# Minimal template (bare-bones)
npx sv create my-app --template minimal

# Library template (publishable Svelte components via @sveltejs/package)
npx sv create my-lib --template library
```

## 4. Skip the install step

```sh
npx sv create my-app --no-install
# later:
cd my-app && npm install
```

## 5. Use a specific package manager

```sh
npx sv create my-app --install npm
npx sv create my-app --install pnpm
npx sv create my-app --install yarn
npx sv create my-app --install bun
```

## 6. Add add-ons to an existing project

```sh
cd my-app
npx sv add prettier
npx sv add eslint
npx sv add vitest
npx sv add playwright
npx sv add tailwindcss
npx sv add drizzle
npx sv add lucia
# multiple at once
npx sv add prettier eslint vitest
```

## 7. TypeScript variants

```sh
# TypeScript syntax (.ts files, tsconfig.json)
npx sv create my-app --types ts

# TypeScript via JSDoc (.js files + jsconfig.json)
npx sv create my-app --types jsdoc

# No TypeScript
npx sv create my-app --types none
```

## 8. Choose the current directory

```sh
# scaffold in the current directory
npx sv create . --template minimal --types ts --no-add-ons --install npm
```

## 9. Check sv CLI version / help

```sh
npx sv --version
npx sv create --help
npx sv add --help
```

## 10. Editor setup

After scaffolding, install the Svelte VS Code extension:

```text
ext install svelte.svelte-vscode
```

Other editors: see <https://sveltesociety.dev/collection/editor-support-c85c080efc292a34>.

## 11. Inspect the generated package.json

```json
{
  "name": "my-app",
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite dev",
    "build": "vite build",
    "preview": "vite preview"
  },
  "devDependencies": {
    "@sveltejs/adapter-auto": "^3.0.0",
    "@sveltejs/kit": "^2.0.0",
    "@sveltejs/vite-plugin-svelte": "^4.0.0",
    "svelte": "^5.0.0",
    "vite": "^5.0.0"
  }
}
```

Note `"type": "module"` — `.js` files are ESM; CommonJS needs `.cjs`.

## 12. Add a custom adapter after scaffold

```sh
npm install -D @sveltejs/adapter-static
# then edit svelte.config.js:
#   import adapter from '@sveltejs/adapter-static';
#   adapter({ pages: 'build', assets: 'build', fallback: 'index.html' });
```