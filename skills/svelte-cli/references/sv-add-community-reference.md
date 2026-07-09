# Community add-ons reference

> Community add-ons are currently **experimental**. The API may change. Don't use them in production yet.
>
> Svelte maintainers have not reviewed community add-ons for malicious code! Use at your discretion.

---

## What is a community add-on?

An npm package that exports an `sv` add-on definition. Anyone can publish one. Users install via `sv add @org/name`.

## Discovery

Tagged on npm with the `sv-add` keyword:

- https://www.npmjs.com/search?q=keywords%3Asv-add
- https://www.npmx.dev/search?q=keyword:sv-add

## Invocation forms

```sh
# Org-name lookup — looks up @org/sv
npx sv add @supacool

# Full package name
npx sv add @supacool/sv
npx sv add @my-org/core

# Pinned version
npx sv add @my-org/sv@1.2.3

# Local add-on (development or internal use)
npx sv add file:../path/to/my-addon

# During create
npx sv create --add eslint @supacool

# Mix official + community
npx sv add eslint @supacool
```

On Windows PowerShell, `@` is a special character — wrap the argument in single quotes:

```powershell
npx sv add '@supacool'
```

## Package naming rules

The CLI only resolves scoped packages — names must begin with `@`. Plain (unscoped) names like `npx sv add my-lib` are rejected.

| Form | Resolves to |
| ---- | ----------- |
| `@org` | `@org/sv` (then `@org/sv@latest`) |
| `@org/sv` | `@org/sv@latest` |
| `@org/sv@1.2.3` | `@org/sv@1.2.3` |
| `@org/whatever` | `@org/whatever` |

## Safety

- Svelte maintainers do **not** audit community add-ons.
- Pass `--no-download-check` to suppress the warning (does not add safety).
- For internal use, `file:` paths let you avoid publishing to npm at all.

## Local development workflow

```sh
# Add-on package
cd ~/work/my-addon
npm run build      # bundles to dist/ via tsdown

# Test in a target project
cd ~/work/test-project
npx sv add file:~/work/my-addon
```

Iterate: edit -> `npm run build` -> re-run `sv add`. The `demo-add` script (added by `npx sv create --template addon`) does the rebuild automatically.

## Warnings shown to the user

| Flag | Effect |
| ---- | ------ |
| `--no-git-check` | Don't warn about uncommitted changes. |
| `--no-download-check` | Don't warn about community add-on downloads. |
| `--install <pm>` | Pre-pick a package manager for install. |
| `--no-install` | Don't prompt to install dependencies. |
| `-C`, `--cwd <path>` | Project root to operate on. |

## Compatibility warning

Users get a warning if their installed `sv` major version doesn't match the peer dependency range specified by the add-on. Specify a minimum compatible `sv` in your `package.json` `peerDependencies` (e.g. `"sv": "^0.13.0"`).

## See also

- `custom-addon-reference.md` — building, testing, bundling, and publishing your own add-on
- `../examples/sv-add.md` — practical invocation examples