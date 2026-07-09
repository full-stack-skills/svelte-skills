# Local MCP setup

The local (`stdio`) MCP server is the `@sveltejs/mcp` npm package. Pick the install pattern that matches your client.

> Base command: `npx -y @sveltejs/mcp`

---

## 1. Claude Code

```bash
# user scope (default)
claude mcp add -t stdio -s user svelte -- npx -y @sveltejs/mcp

# project scope (committed under .mcp.json)
claude mcp add -t stdio -s project svelte -- npx -y @sveltejs/mcp

# local scope (per-worktree, not committed)
claude mcp add -t stdio -s local svelte -- npx -y @sveltejs/mcp
```

`[scope]` must be `user`, `project`, or `local`.

---

## 2. Claude Desktop

Edit `claude_desktop_config.json` (Settings → Developer → Edit Config):

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

Then restart Claude Desktop.

---

## 3. Codex CLI

In `~/.codex/config.toml` (or wherever `[config docs](https://github.com/openai/codex/blob/main/docs/config.md)` points):

```toml
[mcp_servers.svelte]
command = "npx"
args = ["-y", "@sveltejs/mcp"]
```

Or use the Codex Svelte plugin (`codex plugin install svelte`) which sets this up automatically.

---

## 4. Copilot CLI

Interactive (recommended):

```bash
/mcp add
```

Then pick `npx` as the command and `-y @sveltejs/mcp` as the arguments.

Or write `~/.copilot/mcp-config.json` directly:

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

---

## 5. Antigravity CLI

Edit `~/.gemini/config/mcp_config.json`:

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

---

## 6. OpenCode

Interactive:

```bash
opencode mcp add
# pick Local → name: svelte → command: npx -y @sveltejs/mcp
```

Or programmatically — add `@sveltejs/opencode` to your OpenCode config to get the plugin (recommended; see `references/setup-clients-reference.md`).

---

## 7. VS Code

1. Command Palette → `MCP: Add Server...`
2. Pick `Command (stdio)`
3. Type `npx -y @sveltejs/mcp`
4. Name it `svelte`
5. Choose `Global` or `Workspace` scope

Workspace-level MCP entries are saved in `.vscode/mcp.json`:

```json
{
  "servers": {
    "svelte": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@sveltejs/mcp"]
    }
  }
}
```

---

## 8. Cursor

1. Command Palette → `View: Open MCP Settings`
2. Click `Add custom MCP` and paste:

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

Or use `npx cursor-plugin svelte` (Cursor CLI is plugin-incompatible as of writing — use the marketplace in the desktop app: `/add-plugin svelte`).

---

## 9. Zed

Preferred: install the official Svelte MCP Server extension from `zed.dev/extensions/svelte-mcp`.

Manual:

1. Command Palette → `agent: open settings`
2. Find `Model Context Protocol (MCP) Servers`
3. `Add Server` → `Add Custom Server`:

```json
{
  "svelte": {
    "command": "npx",
    "args": ["-y", "@sveltejs/mcp"]
  }
}
```

---

## 10. Verifying a local install

Any of the following confirms the MCP server is wired up:

```bash
# smoke-test the CLI (it runs in CLI mode when given a subcommand)
npx -y @sveltejs/mcp list-sections | head

# Claude Code
claude mcp list

# Codex CLI
codex mcp list
```

If `list-sections` prints a long catalogue of doc sections, the local MCP server is healthy.

---

## 11. Pinning a version

```bash
# global install for a fixed version
npm i -g @sveltejs/mcp@1.2.3

# then reference the binary directly
claude mcp add -t stdio -s user svelte -- svelte-mcp
```

---

## 12. Other clients

For any MCP client not listed above, consult its docs and configure an stdio server with:

- **command**: `npx`
- **args**: `["-y", "@sveltejs/mcp"]`
- (optional) **name**: `svelte`
