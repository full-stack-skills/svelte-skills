# adapter-node

Standalone Node server. Most flexible — works on any Node host (VPS, Docker, Heroku, Fly.io, Render, etc).

## 1. Install & configure

```sh
npm i -D @sveltejs/adapter-node
```

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-node';
export default { kit: { adapter: adapter() } };
```

## 2. Build & run

```sh
npm run build
PORT=4000 node build
```

Defaults: `HOST=0.0.0.0`, `PORT=3000`. Output dir defaults to `build`.

## 3. Deploy with production dependencies

```sh
npm ci --omit dev
node build
```

`devDependencies` are bundled into the output; `dependencies` are NOT — install them on the host.

## 4. Custom server (Express example)

```js
// my-server.js
import { handler } from './build/handler.js';
import express from 'express';

const app = express();

app.get('/healthcheck', (req, res) => res.end('ok'));

app.use(handler);

app.listen(3000, () => console.log('listening'));
```

`handler` is the SvelteKit middleware. Add your own routes BEFORE `app.use(handler)` so they take precedence. Note: server-lifecycle env vars (`PORT`, `HOST`, `SHUTDOWN_TIMEOUT`, etc.) are NOT honored by `handler` — only by the default `node build` server.

## 5. Environment variables

```sh
PORT=4000 \
HOST=127.0.0.1 \
ORIGIN=https://my.site \
node build
```

Behind a reverse proxy, tell SvelteKit about forwarded headers:

```sh
PROTOCOL_HEADER=x-forwarded-proto \
HOST_HEADER=x-forwarded-host \
PORT_HEADER=x-forwarded-port \
ADDRESS_HEADER=True-Client-IP \
XFF_DEPTH=2 \
node build
```

## 6. Graceful shutdown

Listens for `SIGTERM`/`SIGINT`, calls `server.close()`, waits for in-flight, then force-closes after `SHUTDOWN_TIMEOUT` (default 30s). Hook in cleanup:

```js
process.on('sveltekit:shutdown', async (reason) => {
  await db.close();
  // reason is 'SIGTERM' | 'SIGINT' | 'IDLE'
});
```

## 7. Body size limit

Default 512kb. Override with `BODY_SIZE_LIMIT`:

```sh
BODY_SIZE_LIMIT=10M node build
```

Set to `Infinity` to disable, then implement your own check in `hooks.server.js`.

## 8. Compression

Add a reverse proxy in front of Node (nginx, Caddy) for compression. If you must do it in-process, use `@polka/compression`:

```js
import compression from '@polka/compression';
app.use(compression({ threshold: 0 }), handler);
```

Don't use the popular `compression` package — it doesn't support streaming, breaks SvelteKit's streamed responses.

## 9. systemd socket activation

```ini
# /etc/systemd/system/myapp.service
[Service]
Environment=NODE_ENV=production IDLE_TIMEOUT=60
ExecStart=/usr/bin/node /usr/bin/myapp/build
```

```ini
# /etc/systemd/system/myapp.socket
[Socket]
ListenStream=3000

[Install]
WantedBy=sockets.target
```

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now myapp.socket
```

App auto-starts on first request, sleeps after `IDLE_TIMEOUT` seconds idle.

## 10. Custom envPrefix

If `PORT`/`HOST` collide with another process:

```js
adapter({ envPrefix: 'MY_APP_' })
```

```sh
MY_APP_PORT=4000 MY_APP_ORIGIN=https://my.site node build
```

`LISTEN_PID`/`LISTEN_FDS` (systemd) are always read without the prefix.

## See also

- [Building reference](../references/building-reference.md)
- [Adapter comparison](../references/adapters-comparison.md)
