# MCP tools in use

The MCP server exposes four tools. Here is what a typical session looks like and how to call each one.

---

## 1. `list-sections` — discover what's available

```text
> List all Svelte documentation sections.
```
The agent calls:

```json
{ "tool": "list-sections", "args": {} }
```

The tool returns rows like:

```json
{
  "sections": [
    { "title": "$state", "use_cases": "always, any svelte project, core reactivity, state management, counters, forms, todo apps, interactive ui, data updates, class-based components", "path": "svelte/$state" },
    { "title": "$derived", "use_cases": "always, any svelte project, computed values, reactive calculations, derived data, transforming state, dependent values", "path": "svelte/$derived" },
    { "title": "$effect", "use_cases": "canvas drawing, third-party library integration, dom manipulation, side effects, intervals, timers, network requests, analytics tracking", "path": "svelte/$effect" }
  ]
}
```

---

## 2. `get-documentation` — fetch the relevant sections

After scanning `use_cases`, the agent picks the matching paths:

```json
{
  "tool": "get-documentation",
  "args": { "sections": ["svelte/$state", "svelte/$derived"] }
}
```

It also accepts a single section or a comma-separated string (CLI form). Token tip: only request what the `use_cases` strings indicate — pulling the entire catalog is wasteful.

---

## 3. `svelte-autofixer` — the agentic fix loop

When you ask the agent to **write** a `.svelte` file, it must:

1. Draft the component.
2. Call `svelte-autofixer` with the code.
3. Read the response — `{ issues, suggestions, require_another_tool_call_after_fixing }`.
4. Apply every fix.
5. Re-invoke the tool until the response is empty.

Sample response that triggers another loop:

```json
{
  "issues": ["Each block is missing a key — pass a unique `({id}) => ...` key."],
  "suggestions": ["Prefer `$derived` over `$effect` for computed values."],
  "require_another_tool_call_after_fixing": true
}
```

After fixing:

```json
{ "issues": [], "suggestions": [], "require_another_tool_call_after_fixing": false }
```

The agent is now allowed to share the code with you.

---

## 4. `playground-link` — share ephemeral code

Only invoke after the user confirms and only when no project files were written:

```text
> Generate a playground link.
```

```json
{ "tool": "playground-link", "args": { "files": { "App.svelte": "<the final code>" } } }
```

Response:

```json
{ "url": "https://svelte.dev/playground/#H4sIAAAAAAAAA…" }
```

Send the URL back to the user formatted as a markdown link. **Never** call the tool when the code was already written to your repo — the playground duplicates it.

---

## 5. Tool-call sequence (recommended order)

```
list-sections → get-documentation (multiple) → write code → svelte-autofixer (loop) → (optional) playground-link
```

---

## 6. Smart section selection by use-case

Instead of asking for "everything about state":

1. Call `list-sections`.
2. Search the returned rows for `use_cases` strings that contain your keywords (e.g. `forms`, `validation`).
3. Call `get-documentation` with **only** the matching paths.

This keeps the response under the token budget.

---

## 7. Combining multiple sections in one call

```json
{
  "tool": "get-documentation",
  "args": { "sections": ["svelte/$derived", "svelte/$effect", "svelte/$state"] }
}
```

Acceptable inputs:

- Array: `["svelte/$state", "svelte/$effect"]`
- Comma-separated string: `"svelte/$state,svelte/$effect"`
- Title strings: `"$state, What are runes?"` (the resolver falls back to title match)

---

## 8. Tool-call guardrails (also see `AGENTS.md`)

| Guardrail | Why |
|---|---|
| `list-sections` first | Avoids blindly fetching the entire catalog |
| `svelte-autofixer` after every edit | Catches `$effect` vs `$derived`, missing keys, legacy syntax |
| `playground-link` only on opt-in | Prevents accidental long URLs in logs |
| Use `svelte-task` prompt for big tasks | Bundles the contract for the agent |

---

## 9. Handling the `require_another_tool_call_after_fixing` flag

If the flag is `true`, the agent **must** re-invoke `svelte-autofixer` with the updated code. If `false`, you can return the code.

Pseudo-rule for an agent:

```ts
async function writeComponent(code: string) {
  while (true) {
    const result = await svelteAutofixer({ code });
    if (result.issues.length === 0 && result.suggestions.length === 0) return code;
    code = apply(result, code); // apply fixes
    if (!result.require_another_tool_call_after_fixing) return code;
  }
}
```

---

## 10. Quick smoke test from a shell

The same engine is available as a CLI — see `examples/cli-usage.md`:

```bash
npx -y @sveltejs/mcp list-sections | head -5
npx -y @sveltejs/mcp get-documentation 'svelte/$state,svelte/$derived'
```
