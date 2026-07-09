# `sv check` examples

Requires `svelte-check` as a dev dependency:

```sh
npm i -D svelte-check
```

---

## 1. Basic check (human-readable)

```sh
npx sv check
```

Diagnoses the entire project (`.svelte` + `.svelte.ts`/`.svelte.js`, plus TS/JS if a tsconfig is found).

## 2. Watch mode for development

```sh
npx sv check --watch
```

Keeps the process alive and re-runs diagnostics on file save.

## 3. Watch without clearing the screen

```sh
npx sv check --watch --preserveWatchOutput
```

Useful for keeping history visible during long sessions.

## 4. Machine-readable output for CI

```sh
npx sv check --output machine
```

Tab-separated rows: `START`, `ERROR`, `WARNING`, `COMPLETED`, `FAILURE`. See `../references/sv-check-reference.md` for the full schema.

## 5. Verbose machine output (ndjson)

```sh
npx sv check --output machine-verbose > results.ndjson
```

Each row is a JSON object with `start`, `end`, `code`, `source`, and `message`.

## 6. CI gate — fail on warnings

```sh
npx sv check --fail-on-warnings
```

## 7. Only errors, not warnings

```sh
npx sv check --threshold error
```

## 8. Custom compiler warning policy

```sh
# Silence unused CSS selectors but escalate missing a11y attributes
npx sv check --compiler-warnings "css_unused_selector:ignore,a11y_missing_attribute:error"
```

## 9. Disable TS/JS diagnostics

```sh
npx sv check --no-tsconfig
```

Skip `.ts` / `.js` files. Only `.svelte` and its embedded TS are checked.

## 10. Specific tsconfig

```sh
npx sv check --tsconfig ./tsconfig.json
npx sv check --tsconfig ./configs/strict.json
```

Restricts the file set to `files` / `include` / `exclude` from the given config.

## 11. Ignore build output directories

```sh
npx sv check --no-tsconfig --ignore "dist,build,.svelte-kit"
```

Paths are relative to workspace root, comma-separated, quoted.

## 12. Diagnostic source subset

```sh
# Only JS/TS + Svelte (no CSS diagnostics)
npx sv check --diagnostic-sources "js,svelte"

# Only Svelte
npx sv check --diagnostic-sources "svelte"
```

Valid sources: `js` (includes TypeScript), `svelte`, `css`. Default: all three.

## 13. Different workspace path

```sh
npx sv check --workspace ./packages/web
```

## 14. CI pipeline integration

```yaml
# GitHub Actions
- name: svelte-check
  run: npx sv check --output machine --fail-on-warnings | tee check.log
```

Parse `ERROR` / `WARNING` lines for annotations; rely on `COMPLETED` row for the summary count.