---
name: svelte-ai
description: Svelte AI 助手集成技能。当用户为 Claude Code/Cursor/Copilot/Codex/OpenCode/VS Code 等 AI 编程工具配置 Svelte MCP server、安装 svelte-autofixer/list-sections/get-documentation MCP tools、使用 svelte-task 提示、调用 sv CLI（svelte-ai CLI）、或利用 svelte-code-writer/svelte-core-bestpractices 子智能体编写 Svelte 5 代码时使用。
---

# Svelte AI

## When to use this skill

Use this skill when:

- Configuring the **Svelte MCP server** (local stdio via `@sveltejs/mcp` or remote via `https://mcp.svelte.dev/mcp`) for AI coding tools (Claude Code, Claude Desktop, Cursor, VS Code, Codex CLI, Copilot CLI, Antigravity CLI, OpenCode, Zed, GitHub Coding Agent).
- Adding an `AGENTS.md` (or `CLAUDE.md` / `GEMINI.md`) block that describes the four MCP tools (`list-sections`, `get-documentation`, `svelte-autofixer`, `playground-link`).
- Invoking the `svelte-task` prompt, the `doc-section` resource, or any of the four MCP tools in an AI session.
- Running the `@sveltejs/mcp` CLI as `npx @sveltejs/mcp <command>` for scripted documentation lookup (`list-sections` / `get-documentation`) or static-analysis linting (`svelte-autofixer`).
- Installing / loading the `svelte-code-writer` skill (CLI workflow) or the `svelte-core-bestpractices` skill (Svelte 5 runes rules).
- Configuring the `svelte-file-editor` subagent that handles `.svelte` / `.svelte.ts` / `.svelte.js` files in a separate context window.

Skip this skill when the work is pure Svelte 5 syntax questions with no AI tooling involved — use the `svelte` skill instead.

---

## Critical: Svelte AI overview

The Svelte team ships **four coordinated AI tools** that can be used together or individually:

| Tool | Purpose |
|---|---|
| **Instructions** | A small prompt always injected into the agent session to make it aware of the available tools |
| **MCP server** (`@sveltejs/mcp`) | Provides tools, prompts and resources that pull documentation from svelte.dev and statically analyse generated code |
| **Skills** | Lazy-loaded instructions (e.g. `svelte-code-writer`, `svelte-core-bestpractices`) that teach the agent the `@sveltejs/mcp` CLI and Svelte 5 best practices |
| **Subagents** | Focused agents (`svelte-file-editor`) that perform atomic operations in a separate context window |

Quick mental model:

- **MCP server** = live, protocol-based access (the tool runs in your session).
- **Skills** = static markdown the agent reads on demand.
- **Subagents** = delegated workers with isolated context.
- **CLI** = the same engine as the MCP server but driven from your shell.

---

## Critical: AGENTS.md standard

Add this block to `AGENTS.md` (or `CLAUDE.md` / `GEMINI.md`) so the agent knows which MCP tools exist and when to call them. The block is auto-installed by `npx sv add mcp`:

```markdown
You are able to use the Svelte MCP server, where you have access to comprehensive Svelte 5 and SvelteKit documentation. Here's how to use the available tools effectively:

## Available Svelte MCP Tools:

### 1. list-sections
Use this FIRST to discover all available documentation sections. Returns a structured list with titles, use_cases, and paths.
When asked about Svelte or SvelteKit topics, ALWAYS use this tool at the start of the chat to find relevant sections.

### 2. get-documentation
Retrieves full documentation content for specific sections. Accepts single or multiple sections.
After calling the list-sections tool, you MUST analyze the returned documentation sections (especially the use_cases field) and then use the get-documentation tool to fetch ALL documentation sections that are relevant for the user's task.

### 3. svelte-autofixer
Analyzes Svelte code and returns issues and suggestions.
You MUST use this tool whenever writing Svelte code before sending it to the user. Keep calling it until no issues or suggestions are returned.

### 4. playground-link
Generates a Svelte Playground link with the provided code.
After completing the code, ask the user if they want a playground link. Only call this tool after user confirmation and NEVER if code was written to files in their project.
```

Key contract:

1. `list-sections` first → 2. `get-documentation` for relevant sections → 3. `svelte-autofixer` after every code edit → 4. `playground-link` only on user request and only when code was NOT written to project files.

---

## Critical: Setup — local vs remote

Two flavours of the MCP server exist:

| Variant | Command / URL | Best for |
|---|---|---|
| **Local (stdio)** | `npx -y @sveltejs/mcp` | Single-machine, no network, version-pinned via npm |
| **Remote (HTTP)** | `https://mcp.svelte.dev/mcp` | Always-current docs, shared instance, requires HTTPS-supporting client |

### Local setup — AI client configurations

| Client | Command / file |
|---|---|
| **Claude Code** | `claude mcp add -t stdio -s [user\|project\|local] svelte -- npx -y @sveltejs/mcp` |
| **Claude Desktop** | Edit `claude_desktop_config.json` → `{ "mcpServers": { "svelte": { "command": "npx", "args": ["-y", "@sveltejs/mcp"] } } }` |
| **Codex CLI** | Add to `~/.codex/config.toml` → `[mcp_servers.svelte] command = "npx" args = ["-y", "@sveltejs/mcp"]` |
| **Copilot CLI** | `/mcp add` (interactive) or `~/.copilot/mcp-config.json` |
| **Antigravity CLI** | Edit `~/.gemini/config/mcp_config.json` |
| **OpenCode** | `opencode mcp add` → choose `Local` → `npx -y @sveltejs/mcp` |
| **VS Code** | Command Palette → `MCP: Add Server...` → `Command (stdio)` → `npx -y @sveltejs/mcp` |
| **Cursor** | Command Palette → `View: Open MCP Settings` → `Add custom MCP` |
| **Zed** | Install the [Svelte MCP Server extension](https://zed.dev/extensions/svelte-mcp) |

### Remote setup — AI client configurations

| Client | Command / file |
|---|---|
| **Claude Code** | `claude mcp add -t http -s [scope] svelte https://mcp.svelte.dev/mcp` |
| **Claude Desktop** | Settings → Connectors → Add Custom Connector → URL `https://mcp.svelte.dev/mcp` |
| **Codex CLI** | `~/.codex/config.toml` with `experimental_use_rmcp_client = true` + `[mcp_servers.svelte] url = "https://mcp.svelte.dev/mcp"` |
| **Copilot CLI** | `/mcp add` (interactive) or `~/.copilot/mcp-config.json` with `"url": "https://mcp.svelte.dev/mcp"` |
| **Antigravity CLI** | `~/.gemini/config/mcp_config.json` with `"url": "https://mcp.svelte.dev/mcp"` |
| **OpenCode** | `opencode mcp add` → choose `Remote` → URL `https://mcp.svelte.dev/mcp` |
| **VS Code** | Command Palette → `MCP: Add Server...` → `HTTP` → URL `https://mcp.svelte.dev/mcp` |
| **Cursor** | Command Palette → `View: Open MCP Settings` → `Add custom MCP` (URL form) |
| **GitHub Coding Agent** | Repo Settings → Copilot → Coding agent → MCP config with `"type": "http"` |

Plugin / marketplace installs (recommended where available):

- **Claude Code**: `/plugin marketplace add sveltejs/ai-tools` then `/plugin install svelte`
- **OpenCode**: add `@sveltejs/opencode` to `plugin` in OpenCode config
- **Cursor**: `/add-plugin svelte` (Cursor marketplace)
- **Codex CLI**: `codex plugin marketplace add sveltejs/ai-tools` then `/plugins` → install `svelte`
- **Copilot CLI**: `copilot plugin marketplace add sveltejs/ai-tools` then `copilot plugin install svelte@ai-tools`

---

## Critical: MCP tools

Four tools are exposed by the Svelte MCP server:

### `list-sections`
Returns a structured list of every documentation section: `{ title, use_cases, path }`. Call it **first** in any Svelte conversation so the agent knows what to fetch.

### `get-documentation`
Accepts one or many section paths (comma-separated, or an array). After `list-sections`, analyse the `use_cases` strings and pull only the sections that are actually relevant — fetching too many wastes tokens.

### `svelte-autofixer`
Runs static analysis against a Svelte component or module. Returns `{ issues, suggestions, require_another_tool_call_after_fixing }`. Must be called in a loop: fix → autofixer → fix → … until the result is empty.

### `playground-link`
Generates an ephemeral Svelte Playground URL containing the user's code. Call it **only** after the user confirms and **only** when no project files were written. The link encodes the whole project; it is not stored separately and can get long.

---

## Critical: MCP resources

### `doc-section`
Dynamic resource referenced by URI `svelte://<slug>.md`. Includes the `llms.txt`-flattened version of the chosen documentation page. Useful for adding specific knowledge up-front without having the LLM fetch it on its own (e.g. when you know the task needs transitions, attach `svelte://transition.md`).

---

## Critical: MCP prompts

### `svelte-task`
Send this prompt as a user message whenever you need the agent to build a Svelte component or module. It bundles:

- The list of available documentation sections (so the agent does NOT need to call `list-sections` again).
- A hard rule: **every Svelte component/module written MUST go through `svelte-autofixer` in a fix-loop until no issues remain.**
- A rule for `playground-link` — only on confirmation and only when files were not written; the playground entry point must be `App.svelte`.

Pick the prompt, fill in the `<task>` placeholder with the user request, and submit.

---

## Critical: CLI subcommands

`@sveltejs/mcp` doubles as an MCP server **and** a CLI. With a subcommand, the package prints the result directly to stdout instead of speaking MCP.

```bash
npx -y @sveltejs/mcp <command> [options]
```

Available subcommands:

| Subcommand | Purpose | Example |
|---|---|---|
| `list-sections` | List all doc sections | `npx -y @sveltejs/mcp list-sections` |
| `get-documentation <sections>` | Fetch sections by title or path (comma-separated) | `npx -y @sveltejs/mcp get-documentation 'svelte/$state,svelte/await-expressions'` |
| `svelte-autofixer <code_or_path>` | Lint code or a file | `npx -y @sveltejs/mcp svelte-autofixer src/routes/+page.svelte` |

`svelte-autofixer` options: `--svelte-version <4\|5>` (default `5`), `--async` (enable async analysis for Svelte 5). Output: `{ issues, suggestions, require_another_tool_call_after_fixing }`.

> Shell-quote arguments that contain `$` to avoid variable substitution. Pass file paths when possible.

---

## Critical: Subagents

### `svelte-file-editor`
The shipped subagent. Lifecycle:

1. Gather context via `list-sections` → `get-documentation`.
2. Read the target `.svelte` / `.svelte.ts` / `.svelte.js` file.
3. Edit following Svelte 5 best practices.
4. Validate by calling `svelte-autofixer` — fix → re-validate loop.
5. Return summary of changes, autofixer findings, and recommendations.

Falls back to `svelte-code-writer` skill when MCP tools are absent, and to bare `npx @sveltejs/mcp@latest -y --help` when the skill is also absent.

### `svelte-code-writer` (skill)
Pure CLI workflow. **Must be loaded whenever creating / editing / analysing any `.svelte`, `.svelte.ts`, or `.svelte.js` file when the MCP tools are not available.** Workflow:

1. Uncertain syntax? `list-sections` → `get-documentation`.
2. Reviewing / debugging? `svelte-autofixer` on the code.
3. Always validate before finalising.

### `svelte-core-bestpractices` (skill)
Svelte 5 idioms. Load on every Svelte project when asked to write / edit / analyse a component. Covers `$state`, `$derived`, `$effect`, `$props`, `$inspect.trace`, events, snippets, `{#each}` keys, CSS variables, child-component styling, context, async Svelte, and which legacy features to avoid.

---

## Quick Fixes

- **Agent doesn't call the autofixer**: ensure `AGENTS.md` exists at the repo root and contains the contract above.
- **`$` expansion breaks inline code in shell**: pass file paths to `svelte-autofixer` instead of pasting source, or escape `$` as `\$`.
- **`get-documentation` returns too much / eats tokens**: prune to only the section paths matching `use_cases` strings.
- **Codex CLI remote mode fails**: missing `experimental_use_rmcp_client = true` in `config.toml` (required for HTTP MCP).
- **Zed**: prefer installing the official `Svelte MCP Server` extension over manual JSON config.
- **Cursor CLI**: plugins are not yet supported; use the desktop app for `/add-plugin svelte`.
- **GitHub Coding Agent**: only the remote MCP URL is supported; stdio variant is not available in that environment.
- **Antigravity CLI**: MCP config lives under `~/.gemini/config/mcp_config.json`, not `~/.config/`.

---

## Gotchas

- The `playground-link` tool creates a URL that **encodes the entire project in the link** — links can be long and should not be logged in error trackers.
- `get-documentation` is token-expensive; fetch only sections that match the task. Never re-fetch the whole catalog.
- `$derived` is an expression, not a function. Use `$derived.by(() => …)` when you need a function body.
- Effects do **not** run on the server — never wrap their bodies in `if (browser) { ... }`.
- Async Svelte (await expressions, hydratable) requires Svelte 5.36+ **and** `experimental.async: true` in `svelte.config.js`.
- The autofixer treats `$state` on large, never-mutated objects as a perf smell — reach for `$state.raw(...)` for API responses.
- Each block keys must uniquely identify the item — **never use the loop index as the key**.
- Prefer **keyed** `{#each}` blocks. The autofixer calls out unkeyed lists.
- Snippets declared at the top level of a component can be referenced from `<script>` and can even be exported from `<script module>` if they don't reference component state.
- Subagents (`svelte-file-editor`) consume a separate context window — delegate big edit tasks to them so the main agent's context stays small.

---

## FAQ

**Q: AGENTS.md vs CLAUDE.md vs GEMINI.md?**
A: Same idea. `AGENTS.md` is the cross-tool standard (agents.md). Claude Code reads `CLAUDE.md`, Gemini CLI reads `GEMINI.md`. The Svelte contract works identically in each.

**Q: Local vs remote MCP — which should I pick?**
A: Pick local for offline / CI / version pinning. Pick remote for always-current docs and zero install. Both expose identical tools; no code change needed.

**Q: Why does my agent skip `svelte-autofixer`?**
A: Either the tool isn't registered in the MCP client (re-run the install command) or `AGENTS.md` is missing. Codex CLI users: make sure `experimental_use_rmcp_client = true`.

**Q: Can I run the autofixer on a file path?** Yes — `npx -y @sveltejs/mcp svelte-autofixer src/routes/+page.svelte`.

**Q: What's the difference between a skill and a subagent?**
A: Skills are markdown that the agent reads on demand (cheap). Subagents are delegate workers with their own context window (expensive but keeps the main context clean).

**Q: Does `svelte-task` accept arguments?**
A: Yes — replace the `<task>` placeholder in the prompt body with the actual task before submitting it as a user message.

**Q: Is the remote MCP URL stable?** Yes — `https://mcp.svelte.dev/mcp` is the official, always-current endpoint.

**Q: Where do I install the `svelte-file-editor` subagent?** It's auto-shipped with the Claude Code / Codex / Copilot / OpenCode plugins. In OpenCode you can also pin its `model` / `temperature` / `top_p` / `maxSteps` in `~/.config/opencode/svelte.json`.

**Q: Can Svelte AI run on my Svelte 4 project?** Yes — invoke `svelte-autofixer --svelte-version 4`. The other tools (`list-sections`, `get-documentation`) return docs for both versions.

---

## Reference index

- Examples: `examples/README.md`
- Setup matrix: `references/setup-clients-reference.md`
- Tool reference: `references/mcp-tools-reference.md`
- Resource reference: `references/mcp-resources-reference.md`
- Prompt reference: `references/mcp-prompts-reference.md`
- CLI reference: `references/cli-reference.md`
- Subagent reference: `references/subagents-reference.md`
- Architecture: `references/overview-reference.md`
