# CLI usage of `@sveltejs/mcp`

The same package that powers the MCP server is also a normal CLI. When invoked with a subcommand it prints the result to stdout instead of speaking MCP — perfect for scripts, CI, and quick manual checks.

---

## 1. The one-line mental model

```bash
npx -y @sveltejs/mcp <command> [options]
```

Available commands: `list-sections`, `get-documentation`, `svelte-autofixer`.

---

## 2. Discover what's there

```bash
npx -y @sveltejs/mcp list-sections
```

Returns the full catalogue of `{ title, use_cases, path }`. Pipe to your favourite pager:

```bash
npx -y @sveltejs/mcp list-sections | less
npx -y @sveltejs/mcp list-sections | grep 'path: svelte/\$' | head   # just the runes
```

---

## 3. Fetch docs by path or title

```bash
# single section
npx -y @sveltejs/mcp get-documentation 'svelte/$state'

# multiple (comma-separated, no spaces)
npx -y @sveltejs/mcp get-documentation 'svelte/$state,svelte/$derived,svelte/$effect'

# by title
npx -y @sveltejs/mcp get-documentation 'Async Svelte'
```

The resolver accepts both `path` (e.g. `svelte/$state`) and `title` (e.g. `$state`).

---

## 4. Static-analyse a file

```bash
# most useful — pass a path
npx -y @sveltejs/mcp svelte-autofixer src/routes/+page.svelte

# target an older project
npx -y @sveltejs/mcp svelte-autofixer src/lib/Old.svelte --svelte-version 4

# async Svelte (Svelte 5.36+ only)
npx -y @sveltejs/mcp svelte-autofixer src/routes/+page.svelte --async
```

Output (when there's something to fix):

```json
{
  "issues": ["Each block is missing a key — pass a unique `({id}) => ...` key."],
  "suggestions": ["Prefer $derived over $effect for derived values."],
  "require_another_tool_call_after_fixing": true
}
```

When empty:

```json
{ "issues": [], "suggestions": [], "require_another_tool_call_after_fixing": false }
```

---

## 5. Inline-code caveat (escape `$`)

Inline $ runes get eaten by the shell:

```bash
# bad: $state is interpreted as a variable, expands to empty
npx -y @sveltejs/mcp svelte-autofixer '<script>let count = $state(0);</script>'

# good: escape the $
npx -y @sveltejs/mcp svelte-autofixer '<script>let count = \$state(0);</script>'

# best: pass a file path
npx -y @sveltejs/mcp svelte-autofixer ./tmp.svelte
```

---

## 6. Scripted fix loop (bash)

```bash
#!/usr/bin/env bash
set -euo pipefail

file="$1"

while true; do
  out="$(npx -y @sveltejs/mcp svelte-autofixer "$file")"
  issues="$(echo "$out" | jq '.issues | length')"
  suggs="$(echo "$out" | jq '.suggestions | length')"

  if [[ "$issues" -eq 0 && "$suggs" -eq 0 ]]; then
    echo "clean"
    exit 0
  fi

  echo "$out" | jq '{issues, suggestions}'
  echo "Apply fixes manually, then re-run this script."
  exit 1
done
```

Use this in a pre-commit hook: it will block the commit while a file has known autofixer problems.

---

## 7. CI usage (GitHub Actions)

```yaml
- name: Run Svelte autofixer
  run: |
    npx -y @sveltejs/mcp svelte-autofixer src/routes/+page.svelte | tee out.json
    test "$(jq '.issues | length' out.json)" -eq 0
```

A non-zero count of issues fails the job. The script avoids the `$state` shell hazard by only ever passing file paths.

---

## 8. Help & version

```bash
npx -y @sveltejs/mcp --help
npx -y @sveltejs/mcp <command> --help
npx -y @sveltejs/mcp --version
```
