# Advanced routing

Beyond basic `[param]` segments, SvelteKit supports rest params, optional params, matchers, route sorting, and filename encoding.

## 1. Rest parameter — GitHub file viewer

```tree
src/routes/[org]/[repo]/tree/[branch]/[...file]/+page.svelte
```

```js
// +page.js
export function load({ params }) {
  // /sveltejs/kit/tree/main/documentation/docs/04-advanced-routing.md
  // params.file = 'documentation/docs/04-advanced-routing.md'
  return { file: params.file.split('/') };
}
```

Rest params capture everything after, including `/`. They can also be empty (matching the parent route).

## 2. Custom 404 via rest param

```tree
src/routes/marx-brothers/
├ [...path]/+page.js
├ chico/ +page.svelte
├ harpo/ +page.svelte
├ groucho/ +page.svelte
└ +error.svelte
```

```js
// src/routes/marx-brothers/[...path]/+page.js
import { error } from '@sveltejs/kit';
export function load() {
  error(404, { message: 'Not found' });
}
```

Without `[...path]`, `/marx-brothers/karl` wouldn't match any route and the `marx-brothers/+error.svelte` wouldn't render.

## 3. Optional parameter

```tree
src/routes/[[lang]]/home/+page.svelte
```

Matches both `/home` and `/en/home`. Cannot follow a rest parameter — `[...rest]/[[optional]]` is invalid.

## 4. Matcher — restrict param values

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

`/fruits/apple` → matches. `/fruits/rocketship` → falls through to other routes or 404.

## 5. Matcher for integers

```js
// src/params/integer.js
export function match(param) {
  return /^\d+$/.test(param);
}
```

## 6. Multi-matcher

A matcher can constrain both structure and value:

```js
// src/params/hex.js
export function match(param) {
  return /^[0-9a-f]{6}$/i.test(param);
}
```

```tree
src/routes/color/[hex=hex]/+page.svelte
```

## 7. Route sorting priority

Given:

```tree
src/routes/[...catchall]/+page.svelte       # lowest
src/routes/[[a=x]]/+page.svelte             # optional (low when not at end)
src/routes/[b]/+page.svelte                 # single dynamic
src/routes/foo-[c]/+page.svelte             # static + dynamic
src/routes/foo-abc/+page.svelte             # fully static (highest)
```

`/foo-abc` resolves to the fully-static route. `/foo-def` resolves to `foo-[c]`. Sort order:

1. More specific (fewer params) wins
2. Matchers beat unconstrained params
3. `[[optional]]` and `[...rest]` are lowest unless final segment
4. Ties → alphabetical

## 8. Filename encoding — special characters

```tree
src/routes/smileys/[x+3a]-[x+29]/+page.svelte   # /smileys/:-)
src/routes/[x+2f]well-known/...                 # /.well-known/...
src/routes/[u+d83e][u+dd2a]/+page.svelte        # /🤪 (escaped Unicode)
```

| Char | Escape |
|------|--------|
| `\` | `[x+5c]` |
| `/` | `[x+2f]` |
| `:` | `[x+3a]` |
| `*` | `[x+2a]` |
| `?` | `[x+3f]` |
| `[` | `[x+5b]` |
| `]` | `[x+5d]` |
| `#` | `[x+23]` |
| `%` | `[x+25]` |

Useful for `/.well-known/`, filesystems that disallow certain chars, or emoji filenames.

## 9. `reroute` hook for translated URLs

```js
// src/hooks.js
/** @type {import('@sveltejs/kit').Reroute} */
export function reroute({ url }) {
  if (url.pathname === '/de/ueber-uns') return '/de/about';
  if (url.pathname === '/fr/a-propos') return '/fr/about';
}
```

`reroute` runs before route matching. It must be pure (same input → same output) — SvelteKit caches the result.

## 10. `[x+2e]well-known` for `.well-known`

TypeScript struggles with leading-dot folders. Encode `.` as `[x+2e]`:

```tree
src/routes/[x+2e]well-known/security.txt/+server.js
```

## 11. Static + dynamic mix

```tree
src/routes/blog/+page.svelte             # /blog (list)
src/routes/blog/[slug]/+page.svelte      # /blog/hello-world
src/routes/blog/new/+page.svelte         # /blog/new (overrides [slug]='new')
```

`/blog/new` wins because fully-static beats dynamic. Sort order respects this automatically.

## 12. Type-narrowing matchers

```js
// src/params/word.js
/**
 * @param {string} param
 * @returns {param is string}
 */
export function match(param) {
  return /^\w+$/.test(param);
}
```

Then `params.slug` is typed as `string` (not `string | undefined`) in load functions.

## See also

- [Routing reference](../references/routing-advanced-reference.md)
