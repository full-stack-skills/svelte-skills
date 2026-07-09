# MCP prompts in use

The Svelte MCP server ships a single prompt: **`svelte-task`**. It is a user-role message designed to be sent before any Svelte-task work so that the agent does the right things in the right order.

---

## 1. Where to find the prompt

- **Claude Code / Codex CLI / Copilot CLI**: type `/` and search for `svelte-task` (auto-registered by the plugin).
- **Any MCP client**: list prompts via the MCP `prompts/list` method and submit `svelte-task` with the `task` argument.

---

## 2. Anatomy of the prompt

```markdown
You are a Svelte expert tasked to build components and utilities for Svelte developers. If you need documentation for anything related to Svelte you can invoke the tool `get-documentation` with one of the following paths…

<available-docs>
- title: $state, use_cases: …, path: svelte/$state
- title: $derived, use_cases: …, path: svelte/$derived
…
</available-docs>

These are the available documentation sections that `list-sections` will return, you do not need to call it again.

Every time you write a Svelte component or a Svelte module you MUST invoke the `svelte-autofixer` tool providing the code…

If you are not writing the code into a file, once you have the final version of the code ask the user if it wants to generate a playground link…

<task>
[YOUR TASK HERE]
</task>
```

---

## 3. Filling in `<task>`

Replace `[YOUR TASK HERE]` with the actual user request before submitting. Example prompt payload:

```json
{
  "name": "svelte-task",
  "arguments": {
    "task": "Build a Svelte 5 component that displays a list of todos with toggle and delete actions. Use $state for the store and $derived for the count of completed items."
  }
}
```

---

## 4. Claude Code flow

```text
> /svelte-task Build a counter component that resets after reaching 10.
```

Claude Code resolves `/svelte-task`, fills in the `<task>` slot, and sends the prompt as a user message. The agent then:

1. Reads the catalog (no `list-sections` call needed — it's already in the prompt).
2. Calls `get-documentation` for the relevant sections.
3. Writes the component.
4. Runs `svelte-autofixer` in a loop.
5. Optionally generates a `playground-link` (after opt-in).

---

## 5. Codex CLI / Copilot CLI flow

Identical pattern. From an interactive session:

```text
/svelte-task Migrate this Svelte 4 component (uses $: and export let) to Svelte 5 runes.
```

The CLI client resolves the prompt, the agent does the rest.

---

## 6. When NOT to use the prompt

- The task isn't Svelte-related (the prompt just wastes context).
- You're doing quick targeted edits and already know the syntax (skip the prompt).
- You're chaining the agent manually with `@sveltejs/mcp` CLI tools (no MCP available).

---

## 7. Composing the prompt with project context

You can prepend your own instructions before invoking the prompt:

```text
We're using Svelte 5.36 with experimental.async enabled.

/svelte-task Build a `<DataTable>` component that fetches rows via a remote function and supports sorting + filtering.
```

The Claude/Codex/Copilot clients typically concatenate the user's pre-text with the prompt body.

---

## 8. Inline variant (manual copy)

If a client doesn't expose the prompt selector, paste the body verbatim into the chat as a user message and replace `<task>`:

```markdown
<!-- paste the entire prompt body, with the `<task>` placeholder replaced -->
You are a Svelte expert tasked to build… Replace the `<task>` with:

Build a Svelte 5 component that renders a list of users with their avatars, names, and a follow button.
```

This works in any MCP-capable client.
