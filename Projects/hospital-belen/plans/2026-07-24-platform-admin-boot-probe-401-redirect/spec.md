# Spec — Platform admin `/platform/users` still redirects to tenant login (iteration 2)

## Why the previous fix didn't work

The iter-1 fix (interceptor `config.url.startsWith('/platform')` → `/platform/login`) is
correct but addresses the **wrong request**. The redirect is not triggered by a platform call.

Real boot sequence for `https://kairosaid.inksightdev.com/platform/users` (the apex host):

1. `main.ts:46` runs `authStore.initialize()` on **every** page load, unconditionally.
2. → `authService.getMe()` → `apiService.get('/users/me')` (`auth.service.ts:70`).
3. The apex has **no tenant session**, so `/users/me` → **401**.
4. Interceptor: `config.url = '/users/me'` → `isPlatformRequest` = **false** (tenant context).
5. → tries `POST /auth/refresh` → 401 → `clearSession(false)` → `api.service.ts:34`
   `window.location.href = '/login'`.

`/login` on the apex renders the tenant gateway (`base.ts:10-13`) = the "tenant login" the
user sees. This hard `window.location.href` fires from the axios boot probe and **wins the
race against the router**, so the router's correct guard (`router/index.ts:87-91`, which would
send an unauth platform route to `/platform/login`) never gets to run.

**Root cause:** the tenant session probe (`/users/me`) runs on every page — including platform
pages and the tenant-less apex — and a 401 on that probe triggers the global hard redirect to
`/login`. The interceptor's platform-awareness can't help because the offending request is
genuinely a tenant request.

## Files to modify

- `hospital-belen-web/kairosaid/src/services/api.service.ts`
- `hospital-belen-web/kairosaid/src/main.ts`

## Change 1 — session probes must not trigger the global 401 redirect (the bug fix)

`getMe()` is a **session probe**: it already swallows errors and returns `null`
(`auth.service.ts:68-75`, `platformAuth.service.ts:69-76`). A 401 there means "not logged in",
NOT "session died mid-action" — deciding *where* an unauthenticated user goes is the router
guard's job, not the axios layer's.

In `api.service.ts`, exempt the probe endpoints from **both** the refresh branch and the
terminal `clearSession()` redirect — same shape as the existing `/auth/refresh` exemption
(`api.service.ts:45`, using `.endsWith`). A probe 401 must just reject so `getMe()` returns null.

```ts
private isSessionProbe(config?: { url?: string }): boolean {
  return !!config?.url && (config.url.endsWith('/users/me') || config.url.endsWith('/platform/me'))
}
```

Then in the interceptor: if `isSessionProbe(config)`, skip the whole 401 block and fall through
to `return Promise.reject(error)`. Keep tenant/platform mid-session 401s (real API calls)
behaving exactly as today — refresh queue, `clearSession`, `_retry` guard all unchanged.

With this, on the apex `/platform/users`: the `/users/me` 401 rejects silently → the router
guard runs → unauth platform route → `/platform/login`. Correct target, no hard redirect.

## Change 2 — restore the right auth session per context (make it actually usable)

Two problems in `main.ts`:
- `authStore.initialize()` fires a **tenant** `/users/me` probe on platform pages and the apex,
  where it is meaningless (and, pre-Change-1, caused the redirect).
- `platformStore.initialize()` is **never called anywhere**, so even a platform admin with a
  valid `/platform/*` cookie is unauthenticated after any reload → the guard bounces them to
  `/platform/login` on every refresh.

Gate the boot probe on the route context (path prefix is enough; the router isn't mounted yet
so read `window.location.pathname`):

```ts
if (window.location.pathname.startsWith('/platform')) {
  usePlatformStore().initialize()   // probes /platform/me
} else {
  useAuthStore().initialize()       // probes /users/me
}
```

- On `/platform/*`: restores the platform session; a valid platform cookie now survives reload.
- Off `/platform/*` (tenant subdomain or apex): tenant probe as before. On the apex the tenant
  probe still 401s, but Change 1 makes that a silent reject and the guard sends apex → gateway.

Keep the synchronous-localStorage-before-first-nav note that `main.ts:42-44` documents — call
the chosen `initialize()` before `app.mount('#app')`, same as today.

## Edge cases to cover

- **Apex `/platform/users`, no session** → `/platform/login` (the fix target). Verify no
  `/login` hop.
- **Apex `/platform/users`, valid platform cookie** → `platformStore.initialize()` restores it,
  guard allows, page renders. No login bounce on reload.
- **Tenant subdomain, expired session, on a protected app route** → still lands on `/login`
  (router guard), unchanged.
- **Tenant subdomain, valid session** → `/users/me` restores, unchanged.
- **Mid-session real 401** (e.g. `GET /platform/tenants` after platform session expiry) →
  interceptor still redirects to `/platform/login` (iter-1 behavior), because that call is not a
  session probe. Do NOT exempt it.
- **`/auth/refresh` 401** → unchanged (already exempt from the refresh branch).

## Existing patterns to follow

- Endpoint-suffix exemption already exists at `api.service.ts:45` (`.endsWith('/auth/refresh')`)
  — mirror it for the probes.
- Auth-context split (tenant vs platform stores/services/routes) is established; `main.ts` is the
  one boot path that ignored it — this aligns it, same spirit as iter-1's interceptor change.
- Keep the `// ponytail:` comment style already in `api.service.ts`.

## Out of scope

- The iter-1 interceptor platform-branch stays as is — still correct for mid-session platform
  401s. This iteration adds the probe exemption on top of it, does not revert it.

## Tests

`api.service.ts` unit:
- 401 on `/users/me` → no `/auth/refresh` call, no `window.location.href` write, promise rejects.
- 401 on `/platform/me` → same (no redirect).
- 401 on a non-probe tenant call → existing `/auth/refresh` → `/login` path unchanged.
- 401 on a non-probe `/platform/*` call → `/platform/login` (iter-1) unchanged.

`main.ts` is hard to unit-test (module side effects); cover with one integration/e2e assertion
if the harness supports it: loading `/platform/users` unauth resolves to `/platform/login`, not
`/login`. Otherwise a manual check note in test-results is acceptable at this effort.

## Effort: medium — 2 files, additive; core bug fix is the probe exemption in the interceptor.
