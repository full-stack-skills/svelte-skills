# MCP Prompts Reference

Prompts are user-role messages. They bundle an instructional context (here: how to use the Svelte MCP server) so the model always starts from the right baseline.

---

## `svelte-task`

The single prompt shipped with the Svelte MCP server. Send it as a user message before any Svelte-related task.

### Why it exists

Without the prompt, a model needs to:

1. Figure out which tools exist.
2. Run `list-sections` to discover the catalog.
3. Decide which sections to fetch.
4. Remember to call `svelte-autofixer` after every code edit.
5. Ask before generating a `playground-link`.

`svelte-task` bakes all of that into the prompt body, plus the **entire doc catalog** so step 2 is avoided.

### Argument

```ts
{
  task: string   // replaces the `<task>` placeholder
}
```

### Full prompt body

```
You are a Svelte expert tasked to build components and utilities for Svelte developers. If you need documentation for anything related to Svelte you can invoke the tool `get-documentation` with one of the following paths. However: before invoking the `get-documentation` tool, try to answer the users query using your own knowledge and the `svelte-autofixer` tool. Be mindful of how many section you request, since it is token-intensive!

<available-docs>
- title: $state, use_cases: …, path: svelte/$state
- title: $derived, use_cases: …, path: svelte/$derived
… (≈180 sections from `svelte/`, `kit/`, `cli/`, `ai/` namespaces)
</available-docs>

These are the available documentation sections that `list-sections` will return, you do not need to call it again.

Every time you write a Svelte component or a Svelte module you MUST invoke the `svelte-autofixer` tool providing the code. The tool will return a list of issues or suggestions. If there are any issues or suggestions you MUST fix them and call the tool again with the updated code. You MUST keep doing this until the tool returns no issues or suggestions. Only then you can return the code to the user.

This is the task you will work on:

<task>
[YOUR TASK HERE]
</task>

If you are not writing the code into a file, once you have the final version of the code ask the user if it wants to generate a playground link to quickly check the code in it and if it answer yes call the `playground-link` tool and return the url to the user nicely formatted. The playground link MUST be generated only once you have the final version of the code and you are ready to share it, it MUST include an entry point file called `App.svelte` where the main component should live. If you have multiple files to include in the playground link you can include them all at the root.
```

### Hard rules within the prompt

- The agent MUST call `svelte-autofixer` after every code edit and loop until clean.
- `playground-link` MUST be invoked only after user opt-in and **never** if the code was written to project files.
- The playground entry point MUST be `App.svelte`.
- Multiple files go at the root (no nested folders).
- The catalog is already embedded; `list-sections` is unnecessary.

### Submission examples

Claude Code / Codex / Copilot:

```text
> /svelte-task Build a Svelte 5 Todo list with add/remove/toggle.
```

Raw (any MCP client):

```json
{
  "name": "svelte-task",
  "arguments": { "task": "Build a Svelte 5 Todo list with add/remove/toggle." }
}
```

Manual fallback (paste the body into chat, replace `<task>`).

### When NOT to send `svelte-task`

- The task is not Svelte-related.
- You are scripting deterministically via the CLI (no LLM).
- You already pinned a tighter prompt in `AGENTS.md` and the model respects it.
