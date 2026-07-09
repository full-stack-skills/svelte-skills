# Svelte AI — Architecture & Overview

The Svelte team maintains **four coordinated AI tools**. Each can be used individually, but they compose into a stronger pipeline together.

## The four pillars

| Pillar | Role |
|---|---|
| **Instructions** | A small prompt always injected into the agent session so it knows what is available |
| **MCP server** (`@sveltejs/mcp`) | Live protocol access: tools, prompts and resources that fetch Svelte/SvelteKit docs and run static analysis |
| **Skills** | Lazy-loaded markdown that teach the agent `@sveltejs/mcp` CLI usage and Svelte 5 best practices |
| **Subagents** | Focused workers (`svelte-file-editor`) that operate in their own context window |

## AGENTS.md

The recommended way to make the four tools discoverable: include the Svelte contract in `AGENTS.md` (or `CLAUDE.md` / `GEMINI.md`). The contract tells the agent:

1. Always start with `list-sections`.
2. Pull relevant docs via `get-documentation`.
3. Run `svelte-autofixer` after every code edit until clean.
4. Ask before generating a `playground-link` and never when files were already written.

`npx sv add mcp` writes this contract automatically.

## Available MCP server variant

- **Local (stdio)**: `npx -y @sveltejs/mcp` — single-machine, version-pinned, works offline.
- **Remote (HTTP)**: `https://mcp.svelte.dev/mcp` — always-current, zero install, no CLI access (prompt/tool/resource only).

## Plugin marketplaces

`https://github.com/sveltejs/ai-tools` acts as a plugin marketplace for several AI tools. The marketplace ships:

- The remote MCP server (Claude Code, Codex CLI, Copilot CLI).
- The skills (`svelte-code-writer`, `svelte-core-bestpractices`).
- The `svelte-file-editor` subagent.

Plugin installation:

| AI tool | Marketplace add | Plugin install |
|---|---|---|
| Claude Code | `/plugin marketplace add sveltejs/ai-tools` | `/plugin install svelte` |
| OpenCode | add `@sveltejs/opencode` to `plugin` array | n/a (auto) |
| Cursor | `/add-plugin svelte` (desktop only) | n/a (auto) |
| Codex CLI | `codex plugin marketplace add sveltejs/ai-tools` | `/plugins` → install `svelte` |
| Copilot CLI | `copilot plugin marketplace add sveltejs/ai-tools` | `copilot plugin install svelte@ai-tools` |

## When to recommend the subagent

Editing `.svelte`, `.svelte.ts`, or `.svelte.js` files is the canonical "atomic operation" the subagent handles best. Ask the model explicitly to use `svelte-file-editor` for non-trivial edits. Delegating to it keeps the main agent's context window small and pulls documentation + autofixer into the subagent.

## Repository locations

- **MCP server / plugins / skills / subagent definitions**: [github.com/sveltejs/ai-tools](https://github.com/sveltejs/ai-tools)
- **Skills downloads**: [GitHub releases](https://github.com/sveltejs/ai-tools/releases) (filter on the skill name)
- **Source for `tools/skills`**: [github.com/sveltejs/ai-tools/tree/main/tools/skills](https://github.com/sveltejs/ai-tools/tree/main/tools/skills)
- **Svelte docs (pulled by MCP tools)**: [svelte.dev/docs](https://svelte.dev/docs)
