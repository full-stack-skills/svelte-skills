# Setup — Full Matrix for Every AI Client

Two flavours of the Svelte MCP server:

| Variant | Spec | URL / command |
|---|---|---|
| Local (stdio) | Model Context Protocol over stdio | `npx -y @sveltejs/mcp` |
| Remote (HTTP) | Model Context Protocol over HTTP | `https://mcp.svelte.dev/mcp` |

---

## Local setup

Base command: `npx -y @sveltejs/mcp`.

### Claude Code

```bash
claude mcp add -t stdio -s [user|project|local] svelte -- npx -y @sveltejs/mcp
```

The `[scope]` must be `user`, `project`, or `local`. Project scope is committed under `.mcp.json`.

### Claude Desktop

Settings → Developer → **Edit Config** opens `claude_desktop_config.json`. Add:

```json
{
  "mcpServers": {
    "svelte": {
      "command": "npx",
      "args": ["-y", "@sveltejs/mcp"]
    }
  }
}
```

Restart Claude Desktop.

### Codex CLI

`~/.codex/config.toml` (default location; see [Codex config docs](https://github.com/openai/codex/blob/main/docs/config.md) for advanced setups):

```toml
[mcp_servers.svelte]
command = "npx"
args = ["-y", "@sveltejs/mcp"]
```

Preferred path: install the [Codex plugin](https://developers.openai.com/codex/plugins) (recommended).

### Copilot CLI

Interactive:

```bash
/mcp add
```

Or write `~/.copilot/mcp-config.json`:

```json
{
  "mcpServers": {
    "svelte": {
      "command": "npx",
      "args": ["-y", "@sveltejs/mcp"]
    }
  }
}
```

Preferred: install the [Copilot plugin](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-finding-installing).

### Antigravity CLI

`~/.gemini/config/mcp_config.json`:

```json
{
  "mcpServers": {
    "svelte": {
      "command": "npx",
      "args": ["-y", "@sveltejs/mcp"]
    }
  }
}
```

### OpenCode

```bash
opencode mcp add
```

Pick `Local` under the `Select MCP server type` prompt → `npx -y @sveltejs/mcp`.

Preferred: add `@sveltejs/opencode` to your OpenCode `plugin` array — this sets up MCP, the skills, and `svelte-file-editor` automatically.

### VS Code

1. Command Palette → `MCP: Add Server...`
2. Pick `Command (stdio)`
3. Insert `npx -y @sveltejs/mcp`
4. Name: `svelte`
5. Scope: `Global` or `Workspace`

### Cursor

Command Palette → `View: Open MCP Settings` → `Add custom MCP`:

```json
{
  "mcpServers": {
    "svelte": {
      "command": "npx",
      "args": ["-y", "@sveltejs/mcp"]
    }
  }
}
```

Preferred: install via Cursor marketplace (`/add-plugin svelte`).

### Zed

Install the [Svelte MCP Server extension](https://zed.dev/extensions/svelte-mcp).

Manual alternative: Command Palette → `agent: open settings` → find `Model Context Protocol (MCP) Servers` → `Add Server` → `Add Custom Server`:

```json
{
  "svelte": {
    "command": "npx",
    "args": ["-y", "@sveltejs/mcp"]
  }
}
```

### Other clients

For any stdio-supporting MCP client:

- **command**: `npx`
- **args**: `["-y", "@sveltejs/mcp"]`
- **name**: `svelte`

---

## Remote setup

URL: `https://mcp.svelte.dev/mcp`.

### Claude Code

```bash
claude mcp add -t http -s [user|project|local] svelte https://mcp.svelte.dev/mcp
```

Preferred: install the Svelte plugin:

```bash
/plugin marketplace add sveltejs/ai-tools
/plugin install svelte
```

### Claude Desktop

Settings → Connectors → Add Custom Connector → name `svelte`, URL `https://mcp.svelte.dev/mcp`, Add.

### Codex CLI

`~/.codex/config.toml`:

```toml
experimental_use_rmcp_client = true

[mcp_servers.svelte]
url = "https://mcp.svelte.dev/mcp"
```

The `experimental_use_rmcp_client = true` flag is required.

### Copilot CLI

Interactive:

```bash
/mcp add
```

Or write `~/.copilot/mcp-config.json`:

```json
{
  "mcpServers": {
    "svelte": {
      "url": "https://mcp.svelte.dev/mcp"
    }
  }
}
```

### Antigravity CLI

`~/.gemini/config/mcp_config.json`:

```json
{
  "mcpServers": {
    "svelte": {
      "url": "https://mcp.svelte.dev/mcp"
    }
  }
}
```

### OpenCode

```bash
opencode mcp add
```

Pick `Remote` → `https://mcp.svelte.dev/mcp`.

### VS Code

1. Command Palette → `MCP: Add Server...`
2. Pick `HTTP (HTTP or Server-Sent-Events)`
3. URL: `https://mcp.svelte.dev/mcp`
4. Name + scope

### Cursor

Command Palette → `View: Open MCP Settings` → `Add custom MCP`:

```json
{
  "mcpServers": {
    "svelte": {
      "url": "https://mcp.svelte.dev/mcp"
    }
  }
}
```

### GitHub Coding Agent

Repo → Settings → Copilot → **Coding agent** → MCP configuration:

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

Save → **Save MCP configuration**.

### Other clients

Register a remote MCP server at `https://mcp.svelte.dev/mcp` named `svelte`.

---

## Verifying an install

- CLI: `npx -y @sveltejs/mcp list-sections | head`
- Claude Code: `claude mcp list`
- Codex: `codex mcp list`
- HTTP probe:

```bash
curl -fsS https://mcp.svelte.dev/mcp -X POST \
  -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

## Switching local ↔ remote

Re-run the install command, or edit the relevant JSON config (Cursor / VS Code / Antigravity / Copilot). For Codex, change `[mcp_servers.svelte]` between `command = …` and `url = …` (and toggle the experimental flag for remote).
