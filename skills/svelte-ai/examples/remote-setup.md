# Remote MCP setup

The remote MCP server lives at **`https://mcp.svelte.dev/mcp`**. It's always-current with the latest Svelte docs — no install, no `npx`, no version drift.

> Trade-off vs local: no shell access (so no scripted CLI use), but every doc pull is up to the minute.

---

## 1. Claude Code

```bash
claude mcp add -t http -s [user|project|local] svelte https://mcp.svelte.dev/mcp
```

`scope` defaults to `user` if omitted. Or install the full plugin via the marketplace:

```bash
/plugin marketplace add sveltejs/ai-tools
/plugin install svelte
```

The plugin gives you the remote MCP server, the Svelte skills, and the `svelte-file-editor` subagent.

---

## 2. Claude Desktop

1. Settings → Connectors → Add Custom Connector
2. Name: `svelte`
3. Remote MCP server URL: `https://mcp.svelte.dev/mcp`
4. Add

---

## 3. Codex CLI

`~/.codex/config.toml`:

```toml
experimental_use_rmcp_client = true

[mcp_servers.svelte]
url = "https://mcp.svelte.dev/mcp"
```

> `experimental_use_rmcp_client = true` is **required** when targeting a remote URL — without it Codex falls back to stdio-only.

Or use the Codex plugin:

```bash
codex plugin marketplace add sveltejs/ai-tools
/plugins  # inside an interactive session → install svelte
```

---

## 4. Copilot CLI

Interactive:

```bash
/mcp add
```

Or write `~/.copilot/mcp-config.json` directly:

```json
{
  "mcpServers": {
    "svelte": {
      "url": "https://mcp.svelte.dev/mcp"
    }
  }
}
```

Plugin form (recommended):

```bash
copilot plugin marketplace add sveltejs/ai-tools
copilot plugin install svelte@ai-tools
```

---

## 5. Antigravity CLI

Edit `~/.gemini/config/mcp_config.json`:

```json
{
  "mcpServers": {
    "svelte": {
      "url": "https://mcp.svelte.dev/mcp"
    }
  }
}
```

---

## 6. OpenCode

Interactive:

```bash
opencode mcp add
# pick Remote → name: svelte → URL: https://mcp.svelte.dev/mcp
```

Plugin (auto-configures remote + skills + subagent):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["@sveltejs/opencode"]
}
```

Optional override config — `.opencode/svelte.json`:

```json
{
  "$schema": "https://svelte.dev/opencode/schema.json",
  "mcp": { "type": "remote", "enabled": true },
  "subagent": { "enabled": true },
  "skills": { "enabled": true },
  "instructions": { "enabled": true }
}
```

---

## 7. VS Code

1. Command Palette → `MCP: Add Server...`
2. Pick `HTTP (HTTP or Server-Sent-Events)`
3. URL: `https://mcp.svelte.dev/mcp`
4. Name: `svelte`
5. Scope: `Global` or `Workspace`

---

## 8. Cursor

1. Command Palette → `View: Open MCP Settings`
2. `Add custom MCP`:

```json
{
  "mcpServers": {
    "svelte": {
      "url": "https://mcp.svelte.dev/mcp"
    }
  }
}
```

Or install via the Svelte Cursor marketplace plugin:

```text
/add-plugin svelte
```

---

## 9. GitHub Coding Agent

The remote MCP server is the **only** variant supported in the GitHub Coding Agent (no stdio in that env):

1. Repo → Settings → Copilot → Coding agent
2. Edit MCP configuration:

```json
{
  "mcpServers": {
    "svelte": {
      "type": "http",
      "url": "https://mcp.svelte.dev/mcp",
      "tools": ["*"]
    }
  }
}
```

3. Click **Save MCP configuration**.

The `"tools": ["*"]` entry exposes every tool the server offers (`list-sections`, `get-documentation`, `svelte-autofixer`, `playground-link`, prompts and resources).

---

## 10. Verifying a remote install

```bash
curl -fsS https://mcp.svelte.dev/mcp -X POST \
  -H "content-type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | jq
```

The response should enumerate `list_sections`, `get_documentation`, `svelte_autofixer`, `playground_link`. (Exact names follow whatever your client normalises them to.)

---

## 11. Switching between local and remote

- **Claude Code**: rerun the `claude mcp add` command — it overwrites the existing entry by name.
- **Cursor / VS Code**: edit the JSON; save; the client reloads.
- **Codex**: change `[mcp_servers.svelte]` from `command = …` to `url = …` (and toggle the experimental flag).

---

## 12. Other clients

For any MCP client not listed, register a remote MCP server at the URL `https://mcp.svelte.dev/mcp` and name it `svelte`.
