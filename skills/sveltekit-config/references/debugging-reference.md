# Debugging reference

Breakpoint debugging for SvelteKit projects in various IDEs.

## VS Code

### Method 1: Built-in debug terminal

1. `CMD/Ctrl + Shift + P`
2. "Debug: JavaScript Debug Terminal"
3. Run `npm run dev` in the terminal
4. Set breakpoints in any file
5. Trigger via browser

### Method 2: `launch.json`

`.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "command": "npm run dev",
      "name": "SvelteKit: dev",
      "request": "launch",
      "type": "node-terminal"
    },
    {
      "command": "npm run build && npm run preview",
      "name": "SvelteKit: preview",
      "request": "launch",
      "type": "node-terminal"
    },
    {
      "command": "npm run test",
      "name": "Tests",
      "request": "launch",
      "type": "node-terminal"
    }
  ]
}
```

Pick from Run and Debug pane (or `CMD/Ctrl+Shift+D`), press F5.

### Method 3: Compound launch

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "command": "npm run dev",
      "name": "dev",
      "request": "launch",
      "type": "node-terminal"
    }
  ],
  "compounds": [
    {
      "name": "dev + chrome",
      "configurations": ["dev", {
        "type": "chrome",
        "request": "launch",
        "url": "http://localhost:5173",
        "webRoot": "${workspaceFolder}"
      }]
    }
  ]
}
```

This launches the dev server and Chrome simultaneously.

## Chrome / Edge DevTools

### Method 1: Node inspector flag

```sh
NODE_OPTIONS="--inspect" npm run dev
```

Chrome auto-detects the debugger. Click "Open dedicated DevTools for Node.js" (Node logo, top-left).

Or visit `chrome://inspect` and click "Open dedicated DevTools for Node.js".

### Method 2: Source maps (client only)

For client-side debugging, regular browser DevTools work — SvelteKit generates source maps automatically.

## WebStorm / IntelliJ

Built-in support for Svelte:

1. Run → Edit Configurations → `+` → Node.js
2. JavaScript file: `node_modules/vite/bin/vite.js`
3. Application parameters: `dev`
4. Working directory: project root
5. Set breakpoints, click Run in Debug mode

[WebStorm Svelte docs](https://www.jetbrains.com/help/webstorm/svelte.html)

## Neovim

Use `nvim-dap` + `node-debug2-adapter`:

```lua
-- init.lua
local dap = require('dap')
dap.adapters.node = {
  type = 'server',
  host = 'localhost',
  port = '${port}',
  executable = {
    command = 'node',
    args = { 'node_modules/js-debug/js-debug/src/dapDebugServer.js', '${port}' }
  }
}

dap.configurations.javascript = {
  {
    type = 'node',
    request = 'launch',
    name = 'SvelteKit dev',
    program = 'node_modules/vite/bin/vite.js',
    args = { 'dev' },
    cwd = '${workspaceFolder}',
    sourceMaps = true
  }
}
```

Then `:DapContinue` to start debugging.

## Sublime Text

No first-class SvelteKit support. Use browser DevTools for client-side; remote Node debugging is possible but cumbersome.

## Cloud-specific debugging

### Cloudflare

```sh
wrangler dev .svelte-kit/cloudflare/_worker.js
```

### Netlify

```sh
netlify dev
```

### Vercel

```sh
vercel dev
```

Each emulates the production environment more accurately than `vite preview`.

## `@debug` tag (Svelte)

```svelte
<script>
  let { user } = $props();
</script>

{@debug user}
```

Logs to console whenever `user` changes. Dev-only — stripped in production.

## `console` best practices

```js
// Log structured data
console.log({ user, requestId: crypto.randomUUID() });

// Conditional
if (import.meta.env.DEV) console.log('debug:', value);

// Group related logs
console.group('user load');
console.log(user);
console.log(preferences);
console.groupEnd();

// Trace
console.trace('how did we get here?');
```

## Inspect the network

In Chrome DevTools Network tab:

- Sort by Initiator to find `event.fetch` callers
- Right-click → "Save all as HAR with content"
- Filter by `x-sveltekit-` to see SvelteKit internal requests
- In dev mode, SvelteKit adds timing headers via Server-Timing

## Inspect `event.platform`

```js
// +server.js
export async function GET({ platform }) {
  console.log(platform);  // inspect Cloudflare env / Netlify context
  return new Response('ok');
}
```

## Common debug scenarios

### "Why isn't my load function running?"

Add `console.log` at the top of `load`. Check:
- Server vs universal (which one does what?)
- `building` flag — if true, you're in build mode
- `prerender = true` — load may have run during build, not navigation
- `data-sveltekit-reload` on a link → forces full reload (re-runs everything)

### "Why is hydration failing?"

Console will say "hydration mismatch". Common causes:
- `Date.now()` / `Math.random()` — different on server vs client
- `localStorage` / `window` access at top level — use `$effect` or `onMount`
- Browser-only conditionals based on user-agent

### "Why is my cookie not being set?"

Check:
- `path: '/'` is set (SvelteKit 2 requirement)
- `secure: true` requires HTTPS (except localhost)
- Same-origin policy on `sameSite`
- Response went through SvelteKit's `handle` (don't bypass it)

## Source maps in production

```js
// vite.config.js
export default {
  build: { sourcemap: true }
};
```

Upload `.map` files to your error tracker (Sentry, Rollbar, etc.) so production stack traces map back to source.

## See also

- [examples/debugging.md](../examples/debugging.md)
- [VS Code debugging docs](https://code.visualstudio.com/docs/editor/debugging)
