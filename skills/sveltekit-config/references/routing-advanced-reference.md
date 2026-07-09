# Advanced routing reference

Rest params, optional params, matchers, route sorting, and filename encoding.

## Rest parameters `[...rest]`

Match an unknown number of path segments. Available as `params.rest` (string, `/`-separated).

```tree
src/routes/[org]/[repo]/tree/[branch]/[...file]/+page.svelte
```

Request: `/sveltejs/kit/tree/main/documentation/docs/04-advanced-routing.md`

```js
params = {
  org: 'sveltejs',
  repo: 'kit',
  branch: 'main',
  file: 'documentation/docs/04-advanced-routing.md'
}
```

### Empty rest

`src/routes/a/[...rest]/z/+page.svelte` matches:
- `/a/z` (rest is empty)
- `/a/b/z`
- `/a/b/c/z`

Always validate `rest` (e.g. with a matcher) — it can be empty or contain any string.

### Custom 404 with rest

Without a rest route, `/marx-brothers/karl` doesn't match → falls to `+error.svelte` at root, not `marx-brothers/+error.svelte`.

Add `[...path]/+page.js` returning `error(404)` to engage nested error pages:

```js
import { error } from '@sveltejs/kit';
export const load = () => error(404, { message: 'Not found' });
```

## Optional parameters `[[lang]]`

Wrap with double brackets to make the param optional:

```tree
src/routes/[[lang]]/home/+page.svelte
```

Matches both `/home` and `/en/home`. The `lang` param is `undefined` in the first case.

Cannot follow a rest parameter — `[...rest]/[[optional]]` is invalid (params are matched greedily; the optional would always be unused).

## Matching `[name=type]`

A matcher is a function in `src/params/` that returns `true` for valid params:

```js
// src/params/fruit.js
/**
 * @param {string} param
 * @returns {param is ('apple' | 'orange')}
 * @satisfies {import('@sveltejs/kit').ParamMatcher}
 */
export function match(param) {
  return param === 'apple' || param === 'orange';
}
```

```tree
src/routes/fruits/[page=fruit]/+page.svelte
```

`/fruits/apple` → matches. `/fruits/rocketship` → falls through.

### Type narrowing

Matchers can narrow the param type:

```js
/**
 * @param {string} param
 * @returns {param is string}
 */
export function match(param) {
  return /^[a-z]+$/.test(param);
}
```

Then `params.slug` is `string` in `load`, not `string | undefined`.

### Matchers in two places

Matchers run on both the server (during route resolution) and the client (during link interception). Keep them pure and identical.

### Test matchers

`src/params/foo.test.js` for unit tests.

## Route sorting

When multiple routes match, SvelteKit picks by priority:

1. **Specificity** — fewer dynamic segments = higher priority
   - `foo/bar` > `foo/[bar]`
2. **Matchers win** over unconstrained params
   - `[name=type]` > `[name]`
3. **Final-segment `[[optional]]` / `[...rest]`** are lowest priority
   - `x/[[y]]/z` is treated like `x/z` for sorting
4. **Alphabetical** to break ties

Example:

```tree
src/routes/foo-abc/+page.svelte          # fully static (highest)
src/routes/foo-[c]/+page.svelte          # static + 1 dynamic
src/routes/[[a=x]]/+page.svelte          # 1 optional (low)
src/routes/[b]/+page.svelte              # 1 dynamic
src/routes/[...catchall]/+page.svelte    # rest (lowest)
```

`/foo-abc` → first route. `/foo-def` → second. `/bar` → fourth. `/anything/else` → fifth.

## Filename encoding

Some characters can't appear in filenames (`/` on Linux/Mac, `\ / : * ? " < > |` on Windows) or have special meaning (`#`, `%`, `[`, `]`, `(`, `)`).

Use hex escape `[x+nn]` where `nn` is the hex char code:

| Char | Hex | Escape |
|------|-----|--------|
| `\` | `5c` | `[x+5c]` |
| `/` | `2f` | `[x+2f]` |
| `:` | `3a` | `[x+3a]` |
| `*` | `2a` | `[x+2a]` |
| `?` | `3f` | `[x+3f]` |
| `"` | `22` | `[x+22]` |
| `<` | `3c` | `[x+3c]` |
| `>` | `3e` | `[x+3e]` |
| `\|` | `7c` | `[x+7c]` |
| `#` | `23` | `[x+23]` |
| `%` | `25` | `[x+25]` |
| `[` | `5b` | `[x+5b]` |
| `]` | `5d` | `[x+5d]` |
| `(` | `28` | `[x+28]` |
| `)` | `29` | `[x+29]` |

Example: `/smileys/:-)` → `src/routes/smileys/[x+3a]-[x+29]/+page.svelte`

### Unicode escape `[u+nnnn]`

For Unicode characters:

```tree
src/routes/[u+d83e][u+dd2a]/+page.svelte   # /🤪
src/routes/🤪/+page.svelte                 # equivalent
```

Range: `0000` to `10ffff`. No surrogate pairs needed.

### `.well-known` directories

TypeScript struggles with leading-dot folders. Encode `.` as `[x+2e]`:

```tree
src/routes/[x+2e]well-known/security.txt/+server.js
```

## `reroute` hook

Translate URLs before route matching:

```js
// src/hooks.js
/** @type {import('@sveltejs/kit').Reroute} */
export function reroute({ url }) {
  if (url.pathname === '/de/ueber-uns') return '/de/about';
  if (url.pathname === '/fr/a-propos') return '/fr/about';
}
```

- Pure function — same input → same output
- SvelteKit caches the result per URL on the client
- Does NOT change the browser URL or `event.url`
- `params` are derived from the returned pathname
- Async since 2.18

## Combining patterns

Rest + matcher for slug routing:

```js
// src/params/slug.js
export function match(s) {
  return /^[a-z0-9-]+$/i.test(s);
}
```

```tree
src/routes/blog/[year=int]/[slug]/+page.svelte
src/routes/blog/[year=int]/[...rest]/+page.svelte
```

## See also

- [examples/advanced-routing.md](../examples/advanced-routing.md)
- [SvelteKit routing](https://kit.svelte.dev/docs/routing)
