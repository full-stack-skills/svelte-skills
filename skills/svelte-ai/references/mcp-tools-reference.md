# MCP Tools Reference

Four tools are exposed by `@sveltejs/mcp`. The model decides when to call them; your `AGENTS.md` should pin the order.

---

## 1. `list-sections`

**Purpose**: discover every available documentation section.

**Input**: none.

**Output**: `Section[]` where each `Section = { title, use_cases, path }`.

**When to call**: at the start of any Svelte conversation, before fetching docs.

**Token tip**: read the `use_cases` strings to decide which sections to pull — never request the whole catalog at once.

---

## 2. `get-documentation`

**Purpose**: fetch the full (and up-to-date) content of one or many sections directly from `svelte.dev/docs`.

**Input**:

```ts
{
  sections: string[]         // preferred
  // or
  sections: string           // comma-separated
}
```

Each entry may be either a documentation path (`svelte/$state`) or a section title (`$state`). Unresolved entries return an error plus similar matches.

**Output**: an array of long-form markdown sections. Token-costly — fetch only what you need.

**When to call**: right after `list-sections`, scoped by `use_cases`.

---

## 3. `svelte-autofixer`

**Purpose**: static analysis against generated code; returns issues and suggestions so the model can iterate.

**Input**:

```ts
{
  code: string,                  // raw Svelte component/module source
  svelte_version?: 4 | 5,        // default 5
  async?: boolean                // opt-in for Svelte 5.36+ async analysis
}
```

**Output**:

```ts
{
  issues: string[],                                   // blocking problems
  suggestions: string[],                              // non-blocking improvements
  require_another_tool_call_after_fixing: boolean     // false → clean
}
```

**Iterative loop**:

1. Draft code.
2. Call `svelte-autofixer` with the code.
3. If `issues` or `suggestions` is non-empty, fix → call again.
4. Stop when the response is empty AND `require_another_tool_call_after_fixing` is `false`.

**Common findings**:

- `$effect` used where `$derived` would suffice.
- Each block missing a unique key.
- `export let` / `on:click` / `<slot>` legacy syntax in a Svelte 5 codebase.
- Missing cleanup in effects.
- Unnecessary state for non-reactive variables.

---

## 4. `playground-link`

**Purpose**: ephemeral Svelte Playground link that contains your generated code so the user can try it instantly.

**Input**:

```ts
{
  files: Record<string, string>   // fileName → source; entry point MUST be `App.svelte`
}
```

**Output**:

```ts
{ url: string }   // long URL containing the project
```

**Rules**:

- Ask the user before calling; never assume.
- **Do not** call this when code was already written to project files.
- Files must be at the root level of the playground.
- The URL is not stored separately — it is encoded in the link, which can be very long.

---

## Recommended tool-call order

```
list-sections
   ↓
get-documentation(sections matching use_cases)
   ↓
write the component / module
   ↓
svelte-autofixer (loop until clean)
   ↓
(optional, after user confirms) playground-link
```

## Token & cost budget

- `list-sections` is small — safe to call.
- `get-documentation` is **expensive**: pull only relevant sections.
- `svelte-autofixer` is cheap — call in a loop until clean.
- `playground-link` is cheap, but the resulting URL is huge — never paste it into logs.

## Subagent equivalence

When the MCP server isn't available, the same engine is exposed as the `@sveltejs/mcp` CLI:

| MCP tool | CLI subcommand |
|---|---|
| `list-sections` | `npx -y @sveltejs/mcp list-sections` |
| `get-documentation` | `npx -y @sveltejs/mcp get-documentation 'svelte/$state'` |
| `svelte-autofixer` | `npx -y @sveltejs/mcp svelte-autofixer src/foo.svelte` |

See `references/cli-reference.md`.
