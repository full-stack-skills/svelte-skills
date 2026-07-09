# svelte.config.js Examples

Common `svelte.config.js` setups. All snippets assume ESM (`"type": "module"` in `package.json`).

## 1. Minimal (defaults)

```js
// svelte.config.js
/** @type {import('@sveltejs/kit').Config} */
const config = {};
export default config;
```

## 2. Auto adapter + Vite preprocess

```js
import adapter from '@sveltejs/adapter-auto';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

/** @type {import('@sveltejs/kit').Config} */
export default {
  preprocess: vitePreprocess(),
  kit: { adapter: adapter() }
};
```

## 3. Static adapter (SSG / SPA)

```js
import adapter from '@sveltejs/adapter-static';

/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    adapter: adapter({
      pages: 'build',
      assets: 'build',
      fallback: 'index.html'   // remove for pure SSG
    }),
    prerender: { entries: ['*'] }
  }
};
```

## 4. Node adapter with a custom port

```js
import adapter from '@sveltejs/adapter-node';

/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    adapter: adapter({ out: 'build' }),
    alias: {
      $components: 'src/lib/components'
    }
  }
};
```

```sh
PORT=4000 node build
```

## 5. Custom path aliases

```js
/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    alias: {
      $components: 'src/lib/components',
      $stores: 'src/lib/stores',
      $ui: 'src/lib/ui'
    }
  }
};
```

```svelte
<script>
  import Button from '$components/Button.svelte';
  import { user } from '$stores/auth.svelte';
</script>
```

## 6. CSP nonce

```js
/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    csp: {
      mode: 'auto',
      directives: {
        'script-src': ['self']
      }
    }
  }
};
```

```html
<!-- src/app.html -->
<script nonce="%sveltekit.nonce%">console.log('inline')</script>
```

## 7. CSP with hash-mode

```js
/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    csp: {
      mode: 'hash',
      directives: {
        'default-src': ['self']
      }
    }
  }
};
```

## 8. Service worker registration

```js
/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    serviceWorker: {
      register: true   // auto-registers src/service-worker.js
    }
  }
};
```

## 9. Custom `outDir` for the `.svelte-kit` folder

```js
/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    outDir: '.svelte-kit'
  }
};
```

## 10. TypeScript config customization

```js
/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    typescript: {
      config: (cfg) => ({
        ...cfg,
        compilerOptions: {
          ...cfg.compilerOptions,
          strict: true,
          noUncheckedIndexedAccess: true
        }
      })
    }
  }
};
```

## 11. Public path / base path

```js
/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    paths: {
      base: '/my-app',           // app served from /my-app
      assets: '/my-app/static'   // static asset URL prefix
    }
  }
};
```

## 12. Version stamp (for cache busting)

```js
/** @type {import('@sveltejs/kit').Config} */
export default {
  kit: {
    version: {
      name: Date.now().toString(),
      pollInterval: 0   // disable polling; useful in CI
    }
  }
};
```