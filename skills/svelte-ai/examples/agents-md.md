# AGENTS.md / CLAUDE.md / GEMINI.md templates

The Svelte team recommends a tiny contract so the agent knows which MCP tools exist and when to invoke them. The contract is the same regardless of file name: `AGENTS.md` (cross-tool), `CLAUDE.md` (Claude Code), `GEMINI.md` (Gemini CLI).

> Tip: `npx sv add mcp` installs this contract for you inside an existing Svelte project.

---

## 1. Minimal project AGENTS.md

Drop this at the root of a SvelteKit repo:

```markdown
# AGENTS.md

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

---

## 2. Claude Code — `CLAUDE.md`

Claude Code automatically picks up the contract if you save it as `CLAUDE.md` at the repo root. You can also use the `[Claude Code memory imports](https://docs.claude.com/en/docs/claude-code/memory#claude-md-imports)` syntax to fold the Svelte contract in alongside your team's own instructions.

```markdown
# CLAUDE.md

<!-- existing project memory ... -->

@./AGENTS.md  <!-- imports the Svelte contract verbatim -->
```

---

## 3. Gemini CLI — `GEMINI.md`

Identical content, saved as `GEMINI.md`:

```markdown
# GEMINI.md

You are able to use the Svelte MCP server...

## Available Svelte MCP Tools:

### 1. list-sections
…
```

---

## 4. Monorepo — installing the contract per workspace

For a pnpm/turbo monorepo, ship the contract once at the root and reference it from each `apps/*/AGENTS.md`:

```markdown
<!-- apps/web/AGENTS.md -->

@../../AGENTS.md
```

Each AI tool respects file imports differently — Claude Code imports are explicit, Codex and Gemini handle plain `AGENTS.md`. Always include the literal block when in doubt.

---

## 5. Listing every tool with a one-line summary

Useful inside a longer file — lets the agent scan the tool list quickly:

```markdown
| Tool | When |
|---|---|
| `list-sections` | Always first — discover what docs exist |
| `get-documentation` | Right after `list-sections` — fetch only relevant sections |
| `svelte-autofixer` | Every time you write code, in a loop until clean |
| `playground-link` | Only after user opt-in, only when no files were written |
```

---

## 6. Project structure that pairs with AGENTS.md

```
my-svelte-app/
├── AGENTS.md          ← the contract
├── CLAUDE.md          ← same content, for Claude Code
├── GEMINI.md          ← same content, for Gemini CLI
├── svelte.config.js   ← `experimental.async: true` for Svelte 5.36+
├── package.json       ← `@sveltejs/mcp` may be listed as devDep
├── src/
│   ├── app.html
│   ├── lib/
│   └── routes/
│       └── +page.svelte
```

---

## 7. Tightening the contract for subagents

When `svelte-file-editor` is in use, you can tell the main agent NOT to edit `.svelte` files directly:

```markdown
## Delegation rule
When asked to create, edit, or analyse a `.svelte` / `.svelte.ts` / `.svelte.js` file, ALWAYS delegate to the `svelte-file-editor` subagent in a separate context window. Do not call the MCP tools from this main agent.
```

---

## 8. Adding the contract via `sv add`

```bash
npx sv add mcp
```

Non-interactive equivalent (CI-friendly):

```bash
npx sv add mcp --no-tty
```

The CLI writes `AGENTS.md`, registers the `svelte` MCP server in supported configs, and prints any follow-up steps for clients it could not detect.
