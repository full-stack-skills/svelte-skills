# Writing a custom adapter

If no existing adapter fits your target platform, write your own. The builder API gives you everything you need.

## 1. Adapter skeleton

```js
// adapter-myplatform/index.js
/** @param {any} options */
export default function (options) {
  /** @type {import('@sveltejs/kit').Adapter} */
  const adapter = {
    name: 'adapter-myplatform',
    async adapt(builder) {
      // implement
    }
  };
  return adapter;
}
```

`name` and `adapt` are required. `emulate` and `supports` are optional.

## 2. Builder API

Within `adapt(builder)`, you have access to:

- `builder.writeClient(dest)` — write client assets
- `builder.writeServer(dest)` — write server entry
- `builder.writePrerendered(dest)` — write prerendered HTML
- `builder.generateManifest({ relativePath })` — generate SSR manifest
- `builder.getServerDirectory()` — path to the built server entry

## 3. Minimal adapter that bundles a worker

```js
import { fileURLToPath } from 'node:url';
import { mkdir, writeFile, readFile } from 'node:fs/promises';
import { rollup } from 'rollup';

export default function () {
  return {
    name: 'adapter-myworker',
    async adapt(builder) {
      const out = '.myworker';
      await mkdir(out, { recursive: true });

      // 1. write the server entry
      builder.writeServer(out);

      // 2. write prerendered pages
      builder.writePrerendered(out, { fallback: '200.html' });

      // 3. write client + emit manifest
      const manifest = builder.generateManifest({ relativePath: './' });
      builder.writeClient(out);

      // 4. emit the worker script
      const serverFile = fileURLToPath(import.meta.resolve('./handler.js'));
      const handler = await readFile(serverFile, 'utf-8');
      const worker = `
        import { Server } from './server/index.js';
        const manifest = ${JSON.stringify(manifest)};
        const server = new Server(manifest);
        addEventListener('fetch', (event) => {
          event.respondWith(server.respond(event.request, { platform: {} }));
        });
      `;
      await writeFile(`${out}/_worker.js`, worker);

      // 5. bundle it
      await rollup({
        input: `${out}/_worker.js`,
        output: { file: `${out}/bundle.js`, format: 'esm' }
      });
    }
  };
}
```

## 4. Adapter with `emulate` (dev platform shim)

```js
async emulate() {
  return {
    async platform({ config, prerender }) {
      // return object that becomes event.platform during dev/build/preview
      return { env: { MY_KV: myLocalKvShim() } };
    }
  };
}
```

## 5. Adapter with `supports.read` and `supports.instrumentation`

```js
supports: {
  read: ({ config, route }) => {
    // can the given route use `read` from $app/server?
    return true;
  },
  instrumentation: () => {
    // can this adapter load instrumentation.server.js?
    return true;
  }
}
```

Return `false` if your platform can't support the feature, or throw with a descriptive error explaining how to configure the deployment.

## 6. Server entry expectations

The server code you ship must:

1. Import `Server` from `${builder.getServerDirectory()}/index.js`
2. Instantiate with the manifest: `new Server(manifest)`
3. Listen on the platform's event source (HTTP, fetch, etc.)
4. Convert the platform request to a standard `Request`
5. Call `server.respond(request, { getClientAddress, platform })`
6. Return the resulting `Response`
7. Globally shim `fetch` if the platform doesn't provide one (use `@sveltejs/kit/node/polyfills`)

## 7. Output conventions

Put final output under `build/` and intermediate under `.svelte-kit/[adapter-name]/`. The `writeClient`/`writeServer`/`writePrerendered` helpers handle this — just give them the destination.

## 8. Where to look for examples

Existing adapters are the best reference: https://github.com/sveltejs/kit/tree/main/packages

Copy the closest one (e.g. start from `adapter-node` for a Node-based custom target, or `adapter-vercel` for a serverless target) and adapt.

## See also

- [Adapter comparison](../references/adapters-comparison.md)
