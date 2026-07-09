# Subagents & Skills Reference

The Svelte AI toolkit delegates Svelte-specific work to **subagents** (independent context windows) and feeds best-practice context via **skills** (lazy-loaded markdown).

---

## Subagent: `svelte-file-editor`

### Frontmatter

```markdown
---
name: svelte-file-editor
description: Specialized Svelte 5 code editor. MUST BE USED PROACTIVELY when creating, editing, or reviewing any .svelte file or .svelte.ts/.svelte.js module and MUST use the tools from the MCP server or the `svelte-file-editor` skill if they are available. Fetches relevant documentation and validates code using the Svelte MCP server tools.
---
```

### Body

The subagent:

1. States that it is a Svelte 5 expert and that it MUST use the MCP server tools.
2. Falls back to the `svelte-code-writer` skill if MCP tools are absent.
3. Falls back to `npx @sveltejs/mcp@latest -y --help` if the skill is also absent.
4. Knows all three MCP tools by name: `list-sections`, `get-documentation`, `svelte-autofixer`.
5. Follows a fixed workflow (see below).
6. Returns a summary at the end.

### Workflow

1. **Gather context** — call `list-sections` then `get-documentation` for relevant entries.
2. **Read the target file** to understand current state.
3. **Make changes** following Svelte 5 best practices.
4. **Validate** by calling `svelte-autofixer` against the updated code.
5. **Fix any issues** — re-call the autofixer until no issues remain.
6. **Output format**: summary of changes, autofixer findings, recommendations.

### Tool examples referenced in its prompt

- `list-sections`
- `get-documentation` with arguments like `$state`, `$derived`, `$effect`, `$props`, `$bindable`, `snippets`, `routing`, `load functions`.
- `svelte-autofixer` with the actual code.

### Common autofixer findings it acts on

- Use of `$effect` instead of `$derived` for computations.
- Missing cleanup in effects.
- Svelte 4 syntax (`on:click`, `export let`, `<slot>`).
- Missing keys in `{#each}` blocks.

---

## Skill: `svelte-code-writer`

### Frontmatter / name

```markdown
# Svelte 5 Code Writer
```

### Purpose

CLI fallback when the MCP server is unreachable. Provides the same three operations as commands:

- List docs (`npx @sveltejs/mcp list-sections`).
- Get docs (`npx @sveltejs/mcp get-documentation "<sections>"`).
- Autofix (`npx @sveltejs/mcp svelte-autofixer "<code_or_path>"`).

### Workflow

1. **Uncertain about syntax?** `list-sections` → `get-documentation`.
2. **Reviewing / debugging?** `svelte-autofixer` on the code.
3. **Always validate** — `svelte-autofixer` before finalising.

### Options cheat-sheet

| Flag | Default | Effect |
|---|---|---|
| `--async` | `false` | Enable async Svelte mode |
| `--svelte-version` | `5` | Target Svelte version (`4` or `5`) |

### `$` escape reminder

When passing inline code, `$state`, `$derived`, `$effect` must be escaped as `\$state`, `\$derived`, `\$effect` to avoid shell variable expansion. File paths are safer.

### Releases

`svelte-code-writer` is published as a downloadable skill bundle. Find it at the Svelte ai-tools releases page:

> `https://github.com/sveltejs/ai-tools/releases?q=svelte-code-writer`

---

## Skill: `svelte-core-bestpractices`

### Purpose

Distilled Svelte 5 guidance — loaded automatically when the agent is in a Svelte project and asked to write / edit / analyse a component. Covers reactivity, event handling, styling, library integration, and which legacy features to avoid.

### Rules

| Topic | Rule |
|---|---|
| `$state` | Use only for variables that should be reactive. Use `$state.raw(...)` for large read-only blobs (e.g. API responses) to avoid proxy overhead |
| `$derived` | Prefer over `$effect` for computed values. Pass an **expression** (not a function) — use `$derived.by(() => ...)` for function bodies. Writable. Object/array results are returned as-is (not proxied) |
| `$effect` | Escape hatch. Avoid setting state inside an effect. For library integration, prefer `{@attach …}`. For event code, use the handler directly. For external observation, `createSubscriber`. Effects do not run on the server — never wrap in `if (browser)` |
| `$props` | Treat as if they'll change. Derived-from-prop values must be `$derived` |
| `$inspect.trace` | Drop as the first line of an `$effect` / `$derived.by` (or any function they call) to discover which dependency triggered a re-run |
| Events | `onclick={...}` attribute shorthand (with spread support). Use `<svelte:window>` and `<svelte:document>` for window/document listeners — not `onMount` / `$effect` |
| Snippets | Defined in the template with `{#snippet …}`. Instantiated via `{@render …}`. Top-level snippets reachable from `<script>`. Snippets without component state are also reachable (and exportable) from `<script module>` |
| `{#each …}` | **Always keyed**. The key must uniquely identify the item — never the index. Avoid destructuring items you'll later mutate (e.g. via `bind:`) |
| CSS variables | Use `style:--name={value}` to expose JS values to the component's `<style>` block |
| Child styling | Prefer CSS custom properties (`<Child --color="red" />`). Fall back to `:global` only when the child comes from a library |
| Context | Prefer `createContext` over raw `setContext` / `getContext` for type safety. Context-scoped state avoids leaks during SSR — better than module-level state |
| Async Svelte | Requires Svelte 5.36+ and `experimental.async: true` in `svelte.config.js`. Use await expressions + hydratable |
| Avoid legacy | Use runes (`$state`) instead of implicit reactivity; `$derived`/`$effect` instead of `$:`; `$props` instead of `export let` / `$$props` / `$$restProps`; `onclick` instead of `on:click`; snippets + `{@render …}` instead of `<slot>` / `$$slots` / `<svelte:fragment>`; `<Dynamic>` instead of `<svelte:component>`; `import Self` + `<Self>` instead of `<svelte:self>`; classes with `$state` fields instead of stores; `{@attach …}` instead of `use:action`; clsx-style class arrays/objects instead of `class:` directive |

---

## Where they ship

| Component | Auto-shipped with |
|---|---|
| `svelte-file-editor` subagent | Claude Code plugin, Codex plugin, Copilot plugin, OpenCode plugin (`@sveltejs/opencode`), Cursor plugin |
| `svelte-code-writer` skill | Claude Code, Codex CLI, Copilot CLI, OpenCode plugins; or manually in `.claude/skills`, `.copilot/skills`, `.opencode/skills` |
| `svelte-core-bestpractices` skill | Same |

Manual install: download from [GitHub releases](https://github.com/sveltejs/ai-tools/releases?q=svelte-core-bestpractices) or pick the file from [tools/skills](https://github.com/sveltejs/ai-tools/tree/main/tools/skills).

---

## OpenCode configuration

Pin the subagent model / temperature in `~/.config/opencode/svelte.json` (or `.opencode/svelte.json`):

```json
{
  "$schema": "https://svelte.dev/opencode/schema.json",
  "mcp":  { "type": "remote", "enabled": true },
  "subagent": {
    "enabled": true,
    "agents": {
      "svelte-file-editor": {
        "model": "anthropic/claude-sonnet-4",
        "temperature": 1,
        "top_p": 0.7,
        "maxSteps": 20
      }
    }
  },
  "skills":         { "enabled": true },
  "instructions":   { "enabled": true }
}
```

To enable only one skill:

```json
"skills": { "enabled": ["svelte-core-bestpractices"] }
```
