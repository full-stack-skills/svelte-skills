# Auth reference

Authentication and authorization in SvelteKit. Sessions vs tokens, integration points, library options.

## Sessions vs tokens

| | Sessions | JWT tokens |
|---|----------|------------|
| Storage | DB-backed | Self-contained |
| Revocation | Instant | Hard (must rotate secret or blacklist) |
| Per-request cost | DB query | Verify signature only |
| Latency | Higher | Lower |
| Best for | Apps needing instant revocation | Stateless APIs, edge functions |

## Integration pattern

### 1. Hook checks cookie, populates `locals`

```js
// src/hooks.server.js
export async function handle({ event, resolve }) {
  const sessionId = event.cookies.get('session');
  if (sessionId) {
    event.locals.user = await getUserBySession(sessionId);
  }
  return resolve(event);
}
```

### 2. Type `locals` in `app.d.ts`

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface Locals {
      user: User | null;
    }
  }
}
export {};
```

### 3. Use `locals` in `+page.server.js` / `+server.js`

```js
// +page.server.js
export const load = ({ locals, setHeaders }) => {
  if (!locals.user) {
    return redirect(303, '/login');
  }
  return { user: locals.user };
};
```

### 4. Server hooks for full control

```js
// src/hooks.server.js
export async function handle({ event, resolve }) {
  // Block access to /admin/*
  if (event.url.pathname.startsWith('/admin')) {
    if (!event.locals.user?.isAdmin) {
      return redirect(303, '/login');
    }
  }
  return resolve(event);
}
```

## Cookie setup

SvelteKit 2 requires `path: '/'` on all cookie ops:

```js
// Login
cookies.set('session', sessionId, {
  path: '/',
  httpOnly: true,
  secure: true,
  sameSite: 'lax',   // or 'strict' for stricter CSRF protection
  maxAge: 60 * 60 * 24 * 30  // 30 days
});

// Logout
cookies.delete('session', { path: '/' });
```

## CSRF protection

SvelteKit has built-in CSRF protection: it rejects cross-site POST form submissions by default. If you need to disable for an API consumed cross-origin, set `csrf.checkOrigin = false` in `svelte.config.js` — but implement CORS instead.

## Authorization in `load` functions

```js
export const load = async ({ locals, params }) => {
  const post = await db.getPost(params.slug);
  
  if (!post) error(404, { message: 'Not found' });
  
  // Authorization check
  if (post.authorId !== locals.user?.id && !locals.user?.isAdmin) {
    error(403, { message: 'Forbidden' });
  }
  
  return { post };
};
```

## Libraries

### Better Auth (recommended)

Set up via Svelte CLI:

```sh
npx sv add better-auth
```

Modern, batteries-included auth with email/password, OAuth, magic links, sessions.

### Auth.js (formerly NextAuth)

Works with SvelteKit via community adapter. Use `@auth/sveltekit`.

### Lucia (deprecated but reference)

The [Lucia auth guide](https://lucia-auth.com/) is still an excellent reference for session-based auth patterns. The library itself has been deprecated in favor of building your own with the same patterns.

### Roll-your-own

```js
// src/lib/server/auth.js
import { randomBytes, scryptSync, timingSafeEqual } from 'node:crypto';

export function hashPassword(password) {
  const salt = randomBytes(16).toString('hex');
  const hash = scryptSync(password, salt, 64).toString('hex');
  return `${salt}:${hash}`;
}

export function verifyPassword(password, stored) {
  const [salt, hash] = stored.split(':');
  const test = scryptSync(password, salt, 64);
  return timingSafeEqual(Buffer.from(hash, 'hex'), test);
}
```

## OAuth / OIDC

Most providers (Google, GitHub, Auth0) follow the same pattern:

1. Redirect to provider's authorize URL with `state`, `redirect_uri`, `client_id`
2. Provider redirects back with `code`
3. Exchange `code` for `access_token` at provider's token endpoint
4. Fetch user info
5. Create session, set cookie

`arctic` is a good library for this.

## Edge-compatible auth

Cloudflare Workers, Vercel Edge, Netlify Edge don't have Node APIs. Use:

- `crypto.subtle` instead of Node's `crypto`
- HTTP-based session storage (KV, Durable Objects, Upstash Redis)
- JWT (no DB lookup needed)

## Multi-factor auth

Pattern: after password verification, send a one-time code (email or TOTP) and set a "pending 2FA" cookie. Only issue the full session cookie after the code is verified.

## Password reset

1. User submits email → generate a long random token, store `{ tokenHash, userId, expiresAt }` in DB
2. Send email with reset URL containing the unhashed token
3. User clicks link → look up by hash, set new password, delete token

Always hash tokens at rest (same as passwords).

## See also

- [examples/accessibility.md](../examples/accessibility.md) for accessible auth UI
- SvelteKit hooks: `sveltekit-overview` skill
