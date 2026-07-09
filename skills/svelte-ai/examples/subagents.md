# Subagents in use

The Svelte AI toolkit ships two subagents and two skills you load on demand. Use this file as a cookbook for when and how to delegate.

---

## 1. `svelte-file-editor` — the shipped subagent

Frontmatter:

```markdown
---
name: svelte-file-editor
description: Specialized Svelte 5 code editor. MUST BE USED PROACTIVELY when creating, editing, or reviewing any .svelte file or .svelte.ts/.svelte.js module and MUST use the tools from the MCP server or the `svelte-file-editor` skill if they are available. Fetches relevant documentation and validates code using the Svelte MCP server tools.
---
```

Workflow:

1. **Gather context** (if uncertain about Svelte 5): `list-sections` → `get-documentation`.
2. **Read the target file** to understand current state.
3. **Apply edits** following Svelte 5 best practices.
4. **Validate** with `svelte-autofixer` in a fix loop.
5. **Fix any issues** reported by the autofixer, re-validate.
6. **Report**: summary of changes, autofixer findings, recommendations.

Fallback chain when MCP tools are unavailable: skill → `npx @sveltejs/mcp@latest -y --help`.

---

## 2. Explicitly requesting the subagent from your message

When chatting with a model that has the subagent registered, name it explicitly for big edits:

```text
> Use the svelte-file-editor subagent to refactor src/routes/+page.svelte to use Svelte 5 runes.
```

The main agent forks a fresh context window for the subagent. The subagent fetches docs and runs the autofixer; the main agent sees only the summary.

---

## 3. Letting the model auto-delegate

In your `AGENTS.md`:

```markdown
## Delegation rule
When asked to create, edit, or analyse a `.svelte` / `.svelte.ts` / `.svelte.js` file, ALWAYS delegate to the `svelte-file-editor` subagent in a separate context window.
```

Now any reference to Svelte files automatically routes through the subagent.

---

## 4. `svelte-code-writer` skill — CLI workflow

Loaded when MCP tools aren't available or as a backup. Workflow:

```bash
# 1. discover docs
npx @sveltejs/mcp list-sections

# 2. fetch what you need
npx @sveltejs/mcp get-documentation "$state,$derived,$effect"

# 3. lint the code
npx @sveltejs/mcp svelte-autofixer ./src/lib/Component.svelte
```

Always lint before finalising. Remember to **escape `$` to `\$`** when inline-coding via the shell.

---

## 5. `svelte-core-bestpractices` skill — rules cheat-sheet

Loaded automatically inside a Svelte project. Distilled rules:

| Topic | Rule |
|---|---|
| `$state` | Reserve for **reactive** vars only. Use `$state.raw` for large immutable blobs (e.g. API responses) |
| `$derived` | For computed values, never `$effect`. Pass an expression; use `$derived.by(() => …)` for functions. Writable. Object/array results are NOT proxied |
| `$effect` | Escape hatch. Avoid updating state in effects. Use `{@attach …}` for library integration, event handlers directly, `createSubscriber` for external observation |
| `$props` | Treat as if they will change. Wrap derived-from-prop values in `$derived` |
| `$inspect.trace` | Drop inside `$effect` / `$derived.by` to discover which dep fired |
| Events | `onclick={…}` attribute shorthand; use `<svelte:window>` / `<svelte:document>` for window/document |
| Snippets | Declared in template, referenced via `{@render …}`. Top-level snippets are reachable from `<script>` and exportable from `<script module>` |
| `{#each …}` | **Keyed**, with a unique identifier (NOT the index). Avoid destructuring when mutating items |
| CSS variables | Use `style:--name={value}` to expose JS values to the component's `<style>` block |
| Child styling | Prefer CSS custom properties (`--color="red"`); fall back to `:global` for third-party children |
| Context | Prefer `createContext` over raw `setContext`/`getContext` (type safety). Avoid module-level state for per-user/per-request data |
| Async Svelte | Svelte 5.36+ requires `experimental.async: true` in `svelte.config.js`. Await expressions + hydratable |
| Avoid legacy | `$state` not `let count = 0; count += 1`; `$derived` / `$effect` not `$:`; `$props` not `export let`; `onclick` not `on:click`; snippets not `<slot>`; `<Dynamic>` not `<svelte:component>`; `import Self` not `<svelte:self>`; classes with `$state` not stores; `{@attach …}` not `use:action` |

---

## 6. Custom OpenCode subagent config

In `~/.config/opencode/svelte.json`:

```json
{
  "$schema": "https://svelte.dev/opencode/schema.json",
  "mcp": { "type": "remote", "enabled": true },
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
  "skills": {
    "enabled": ["svelte-core-bestpractices"]
  },
  "instructions": { "enabled": true }
}
```

This pins the model and limits each subagent invocation to 20 steps.

---

## 7. When to pick MCP tools vs CLI vs skill

| Situation | Reach for |
|---|---|
| MCP server registered | MCP tools (live, up-to-date) |
| No MCP server (CI, scripted) | `@sveltejs/mcp` CLI |
| Subagent isolation needed | `svelte-file-editor` |
| LLM prompt-level guidance only | `svelte-core-bestpractices` skill |

---

## 8. A multi-file edit routed through the subagent

```text
> Use the svelte-file-editor subagent to:
>   1. Convert src/lib/Header.svelte to Svelte 5 runes (currently uses export let and $:).
>   2. Convert src/lib/Footer.svelte the same way.
>   3. Replace `class:active` directives with arrays in src/routes/+layout.svelte.
> Report a single summary when done.
```

The subagent runs all three edits with autofixer validation. The main agent consumes only the summary — context stays lean.

---

## 9. Monitoring subagent cost

Because subagents run their own context window, they consume tokens you may not see directly. Watch for:

- Long-running subagents (`maxSteps` set in OpenCode caps them).
- Subagents that re-fetch the doc catalog — the MCP prompt `svelte-task` avoids this by embedding the catalog directly.

---

## 10. Disable subagents when scripting

For deterministic scripts and CI, disable subagents entirely and use the CLI directly:

```bash
# CI-friendly: never spawns a subagent
npx -y @sveltejs/mcp svelte-autofixer src/lib/Header.svelte
```

The CLI never forks a context, so results stay reproducible.
