# References

Quick lookup tables and API summaries for SvelteKit advanced features. Each file focuses on a single topic with key signatures, behavior rules, and edge cases.

| File | Topic |
|------|-------|
| `state-management.md` | SSR state rules, context, URL state, snapshots, derived vs const |
| `remote-functions.md` | query / query.batch / query.live / form / command / prerender API |
| `env-vars.md` | $env/* vs $app/env/* (legacy + explicit), prefixes, static vs dynamic |
| `hooks.md` | handle / sequence / handleFetch / handleError / reroute / transport / init |
| `errors.md` | error() / redirect() / +error.svelte / App.Error / handleValidationError |
| `link-options.md` | data-sveltekit-preload-data/code, reload, replacestate, keepfocus, noscroll |
| `service-worker.md` | $service-worker exports, cache strategies, manual registration |
| `server-only.md` | .server suffix, $lib/server, $env/private, $app/server |
| `app-modules.md` | $app/forms / navigation / state / paths / server / environment / types |
| `shallow-snapshots.md` | snapshot API, pushState/replaceState, page.state, history entries |