# `sv check` reference

`sv check` wraps `svelte-check` and surfaces type / compile / a11y diagnostics for `.svelte`, `.svelte.ts`, `.svelte.js` and (when a tsconfig is found) `.ts` / `.js` files.

Requires:

```sh
npm i -D svelte-check
```

Requires Node 16+.

```
npx sv check [options]
```

---

## CLI options

| Flag | Description |
| ---- | ----------- |
| `--workspace <path>` | Path to your workspace. All subdirectories except `node_modules` and `--ignore` are checked. |
| `--output <format>` | Output format. See [machine-readable output](#machine-readable-output). One of `human`, `human-verbose`, `machine`, `machine-verbose`. |
| `--watch` | Keep the process alive, re-check on file change. |
| `--preserveWatchOutput` | Don't clear the screen in `--watch` mode. |
| `--tsconfig <path>` | Path to a `tsconfig.json` or `jsconfig.json`. Restricts the file set to its `files` / `include` / `exclude`. Enables TS/JS diagnostics. |
| `--no-tsconfig` | Skip `.ts` / `.js` files entirely. Only `.svelte` and embedded TS are checked. |
| `--ignore <paths>` | Comma-separated paths to ignore (relative to workspace root). Only takes effect with `--no-tsconfig`. |
| `--fail-on-warnings` | Exit with non-zero on warnings. |
| `--compiler-warnings <pairs>` | Comma-separated `code:behaviour` pairs. Behaviour is `ignore` or `error`. |
| `--diagnostic-sources <sources>` | Comma-separated list of sources to diagnose. Default: `js,svelte,css`. |
| `--threshold <level>` | `warning` (default; both errors and warnings shown) or `error` (errors only). |

---

## `--compiler-warnings` format

```
--compiler-warnings "code:behaviour[,code:behaviour...]"
```

- `code` — a [compiler warning code](https://svelte.dev/docs/svelte/compiler-warnings) (e.g. `css_unused_selector`, `a11y_missing_attribute`).
- `behaviour` — `ignore` (silence) or `error` (escalate to error).

```sh
npx sv check --compiler-warnings "css_unused_selector:ignore,a11y_missing_attribute:error"
```

Multiple pairs are comma-separated. The whole argument is quoted.

---

## `--diagnostic-sources` values

| Source | Includes |
| ------ | -------- |
| `js` | TypeScript and JavaScript files (via tsconfig). |
| `svelte` | `.svelte` components and `.svelte.ts` / `.svelte.js` modules. |
| `css` | CSS inside `.svelte` files and standalone `.css` files. |

Default: all three. Disable subsets with comma-separated lists:

```sh
npx sv check --diagnostic-sources "js,svelte"     # skip CSS
npx sv check --diagnostic-sources "svelte"        # Svelte only
```

---

## `--threshold`

| Value | Effect |
| ----- | ------ |
| `warning` (default) | Both errors and warnings are shown. |
| `error` | Only errors are shown; warnings hidden. |

Use `error` for stricter CI gates; use `warning` for full visibility.

---

## `--ignore` and `--tsconfig` interaction

| Combination | Behaviour |
| ----------- | --------- |
| `--no-tsconfig --ignore "dist,build"` | Only `.svelte` files; ignores `dist`, `build`. |
| `--tsconfig tsconfig.json` | File set determined by config's `files` / `include` / `exclude`. `--ignore` only affects the watcher. |
| `--tsconfig foo.json --ignore "bar"` | `--ignore` does **not** affect diagnosed files — only what the watcher tracks. |

---

## `--output` formats

### `human`
Default. Colour-coded diagnostics grouped by file, then by line.

### `human-verbose`
Same as `human` plus source / code / position detail.

### `machine`
Single space-separated rows. Designed for CI.

```
1590680325583 START "/path/to/workspace"
1590680326283 ERROR "codeaction.svelte" 1:16 "Cannot find module 'blubb'..."
1590680326778 WARNING "imported-file.svelte" 0:37 "Component has unused export..."
1590680326807 COMPLETED 20 FILES 21 ERRORS 1 WARNINGS 3 FILES_WITH_PROBLEMS
1590680328921 FAILURE "Connection closed"   # only on runtime error
```

### `machine-verbose`
ndjson lines prefixed by timestamp:

```
1590680326283 {"type":"ERROR","filename":"codeaction.svelte","start":{"line":1,"character":16},"end":{"line":1,"character":23},"message":"Cannot find module 'blubb'...","code":2307,"source":"js"}
1590680326778 {"type":"WARNING","filename":"imported-file.svelte","start":{"line":0,"character":37},"end":{"line":0,"character":51},"message":"Component has unused export property...","code":"unused-export-let","source":"svelte"}
```

---

## Row schemas

| Row | Columns (`machine`) | Notes |
| --- | ------------------- | ----- |
| `START` | `timestamp type "workspace"` | Always first. |
| `ERROR` / `WARNING` | `timestamp type "filename" line:col "message"` | Filename is relative to workspace. |
| `COMPLETED` | `timestamp type files errors warnings files_with_problems` | Always last. |
| `FAILURE` | `timestamp type "message"` | Only on runtime error. |

`machine-verbose` replaces the trailing columns with a JSON object (see above).

---

## Troubleshooting

See the upstream [language-tools docs](https://github.com/sveltejs/language-tools/blob/master/docs/README.md) for preprocessor setup details.

Common issues:

- **`sv check` not found** — install `svelte-check` first.
- **No `.ts`/`.js` diagnostics** — provide `--tsconfig` or pass `--no-tsconfig` to make intent explicit.
- **`--ignore` not honoured** — needs `--no-tsconfig`. With `--tsconfig`, use the config's `exclude`.
- **Empty output in CI** — confirm the working directory is the project root, or pass `--workspace`.

---

## FAQ

**Why no "check only staged files" option?**

`svelte-check` needs to 'see' the whole project graph. A renamed component prop is invisible to a single-file check unless the consumers are also re-checked. The whole-project model catches cross-file regressions that incremental checks miss.

**Why does `--watch` clear the screen?**

`chokidar`/node-tap behaviour — re-running the suite displays only the latest result. Use `--preserveWatchOutput` to keep the history visible.