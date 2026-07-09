# MCP Resources Reference

Resources are LLM-readable context attached to the session by **the user** (not the model). They're useful when you already know the conversation will need a specific piece of documentation — include the resource up-front and avoid the round-trip of asking the LLM to fetch it.

---

## `doc-section`

Dynamic resource backed by every doc page on svelte.dev. Attach a resource to your session to inject that exact doc page into the agent's context.

### URI scheme

```
svelte://<slug>.md
```

Where `<slug>` is the page slug — for example `svelte-transition`, `svelte-$state`, `kit-routing`.

The returned resource is the `llms.txt`-flattened version of the page (long-form markdown, no surrounding chrome), suitable for ingestion by the model.

### Use cases

- You know the task needs **transitions** → attach `svelte://svelte-transition.md`.
- You know you'll be writing **load functions** → attach `svelte://kit-load.md`.
- The agent is going to handle **runes** semantics → attach `svelte://svelte-what-are-runes.md`.

Pre-loading narrows the catalog the model needs to reason about and reduces wasted tool calls.

### How to attach (varies per client)

| Client | How |
|---|---|
| Claude Desktop / Claude Code | `add_resource` / `@<uri>` in the chat context |
| Codex CLI | `codex mcp resources` then add |
| VS Code | Add via the MCP tree |
| MCP spec | `resources/read` call |

### Diff vs tools

| Aspect | Resources (`doc-section`) | Tools (`get-documentation`) |
|---|---|---|
| Pulled by | the user | the model |
| Decision surface | known up-front | decided at runtime |
| Use when | you can predict what docs matter | the model needs to discover |

### Resolution detail

The slug mirrors the URL slug on svelte.dev (e.g. `/docs/svelte/$state` → `svelte://svelte-$state.md`, kebab-case). When in doubt, run `list-sections` to enumerate the canonical paths.
