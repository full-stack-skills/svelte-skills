# Debugging

Breakpoint debugging for SvelteKit client + server code.

## 1. VS Code — built-in debug terminal

1. `CMD/Ctrl + Shift + P`
2. "Debug: JavaScript Debug Terminal"
3. Run `npm run dev` in that terminal
4. Set breakpoints in `.svelte`, `.js`, `.ts` files
5. Trigger via the browser — breakpoints hit

## 2. VS Code — `launch.json`

`.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "command": "npm run dev",
      "name": "dev",
      "request": "launch",
      "type": "node-terminal"
    },
    {
      "command": "npm run build && npm run preview",
      "name": "preview",
      "request": "launch",
      "type": "node-terminal"
    },
    {
      "command": "npm run test",
      "name": "test",
      "request": "launch",
      "type": "node-terminal"
    }
  ]
}
```

Pick a config from the Run and Debug pane and press F5.

## 3. Chrome DevTools — Node inspector

```sh
NODE_OPTIONS="--inspect" npm run dev
```

Then in Chrome:
- DevTools opens automatically
- Or click the Node.js icon (top-left)
- Or visit `chrome://inspect` → "Open dedicated DevTools for Node.js"

This only debugs **client-side** source via source maps.

## 4. Edge / browser DevTools for client code

For client-only debugging, use the regular browser DevTools — source maps work out of the box for SvelteKit.

Set breakpoints in:
- `.svelte` components
- `.js` / `.ts` files
- Inline via the Sources tab

## 5. `console.log` with structured data

```js
console.log({ user, requestId, time: Date.now() });
```

Better than `console.log(user)` — preserves context.

## 6. Server-only breakpoint

Add `debugger;` in `+page.server.js` or `+server.js`:

```js
export async function POST({ request }) {
  debugger;
  // ...
}
```

Combined with the Node inspector setup above, this breaks into DevTools.

## 7. `@debug` in Svelte templates

```svelte
<script>
  let { user } = $props();
</script>

{@debug user}
```

Logs the value to the console and updates when it changes. Dev-only — stripped in production builds.

## 8. Inspect the network

- `event.fetch` calls show up under Server-timed in DevTools
- `handle` hook headers are visible
- SvelteKit adds `x-sveltekit-*` debug headers in dev

## 9. Debug environment-specific issues

Test against the actual platform's CLI:

```sh
wrangler dev       # Cloudflare
netlify dev        # Netlify
vercel dev         # Vercel
```

These emulate the production environment more accurately than `vite preview`.

## 10. Source maps in production

Verify source maps are generated:

```js
// vite.config.js
export default {
  build: { sourcemap: true }
};
```

Then upload them to your error tracker (Sentry, etc.) so stack traces map back to original source.

## See also

- [Debugging reference](../references/debugging-reference.md)
