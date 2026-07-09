# CLI Reference — `@sveltejs/mcp`

The npm package doubles as a CLI. With a subcommand it prints results to stdout (instead of speaking MCP). Useful for scripts, CI, and quick local checks.

## Synopsis

```bash
npx -y @sveltejs/mcp <command> [options]
```

## Top-level flags

```bash
npx -y @sveltejs/mcp --help        # list commands and global options
npx -y @sveltejs/mcp --version     # print the installed version
```

## Commands

| Command | Purpose |
|---|---|
| `list-sections` | List every available doc section (`{ title, use_cases, path }`) |
| `get-documentation <sections>` | Fetch full docs for one or many sections |
| `svelte-autofixer <code_or_path>` | Run static analysis |

### `list-sections`

```bash
npx -y @sveltejs/mcp list-sections
```

Outputs the structured catalog (matching what the MCP `list-sections` tool returns). Useful for piping into greps and into shell loops:

```bash
npx -y @sveltejs/mcp list-sections | grep 'path: svelte/\$' | wc -l
```

### `get-documentation`

```bash
# single section (by path)
npx -y @sveltejs/mcp get-documentation 'svelte/$state'

# multiple (comma-separated)
npx -y @sveltejs/mcp get-documentation 'svelte/$state,svelte/$derived,svelte/$effect'

# by title
npx -y @sveltejs/mcp get-documentation 'Async Svelte'
```

- Each entry can be matched by title or by documentation path.
- If a section cannot be resolved, the CLI returns an error plus similar matches.
- The output is large — request only what you need.

### `svelte-autofixer`

```bash
# by file path (recommended)
npx -y @sveltejs/mcp svelte-autofixer src/routes/+page.svelte

# inline code (escape `$`!)
npx -y @sveltejs/mcp svelte-autofixer '<script>let count = \$state(0);</script>'

# Svelte 4 project
npx -y @sveltejs/mcp svelte-autofixer ./src/lib/Old.svelte --svelte-version 4

# Async Svelte (5.36+)
npx -y @sveltejs/mcp svelte-autofixer ./src/routes/+page.svelte --async
```

#### Options

| Flag | Default | Description |
|---|---|---|
| `--svelte-version <4\|5>` | `5` | Choose which Svelte version to validate against |
| `--async` | `false` | Enable async-Svelte analysis for Svelte 5 |

#### Output

```ts
{
  issues: string[],
  suggestions: string[],
  require_another_tool_call_after_fixing: boolean
}
```

The shape matches the MCP tool output, so a fix-loop script is identical to the in-session agentic loop:

```bash
out=$(npx -y @sveltejs/mcp svelte-autofixer src/foo.svelte)
[ "$(echo "$out" | jq '.issues | length')" = 0 ] || exit 1
```

## Shell hazard

Arguments containing `$` (e.g. `$state`) are interpreted by most shells as variable references. Either:

- Pass file paths (no `$` in the argument), OR
- Escape with `\$`, OR
- Wrap in single quotes and escape inside.

Prefer file paths in CI.

## Help per command

```bash
npx -y @sveltejs/mcp <command> --help
```

## Common scripted patterns

### Fix-loop in bash

```bash
#!/usr/bin/env bash
set -euo pipefail
file="$1"

while true; do
  res="$(npx -y @sveltejs/mcp svelte-autofixer "$file")"
  i=$(echo "$res" | jq '.issues | length')
  s=$(echo "$res" | jq '.suggestions | length')
  [[ "$i" -eq 0 && "$s" -eq 0 ]] && { echo clean; exit 0; }
  echo "issues=$i suggestions=$s"
  echo "$res" | jq '.issues, .suggestions'
  exit 1
done
```

### GitHub Actions

```yaml
- uses: actions/setup-node@v4
  with: { node-version: 22 }
- run: npx -y @sveltejs/mcp svelte-autofixer src/routes/+page.svelte
```

### Pre-commit hook (.git/hooks/pre-commit)

```bash
for f in $(git diff --cached --name-only -- '*.svelte' '*.svelte.ts' '*.svelte.js'); do
  npx -y @sveltejs/mcp svelte-autofixer "$f" >/dev/null
done
```

### Docker

```dockerfile
RUN npx -y @sveltejs/mcp svelte-autofixer ./src/routes/+page.svelte
```
