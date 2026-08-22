# Spec — Platform admin 401 redirects to tenant `/login` instead of `/platform/login`

## Root cause

The global axios 401 interceptor in `hospital-belen-web/kairosaid/src/services/api.service.ts`
is **auth-context blind**. It hard-redirects every 401 to the tenant login `/login` and
tries the tenant refresh endpoint `/auth/refresh` — even for platform-admin requests.

The platform area is a separate auth domain: own store (`platform.store.ts`), own endpoints
(`/platform/login`, `/platform/me`, `/platform/logout`), own login page `/platform/login`.
Any 401 on a `/platform/*` call (expired platform session, `getMe()` on an unauthenticated
admin, a racey call right after `/platform/login`) runs this chain:

`api.service.ts:60` `POST /auth/refresh` → 401 → `clearSession()` → `api.service.ts:30`
`window.location.href = '/login'`

So the admin is yanked out of `/platform/*` onto the tenant login. The router guard itself
is correct (`router/index.ts:87-93` sends unauth platform routes to `/platform/login`); the
hard redirect happens **below** the router, in the axios layer, and wins.

The `/login` exemption (`pathname !== '/login'`) does not cover `/platform/login`, so the
platform login page is not protected from this either.

## Files to modify

- `hospital-belen-web/kairosaid/src/services/api.service.ts` (only file)

## Change — make the interceptor auth-context aware

Decide context from the **request url**, not global state. A request is "platform" when
`config.url` starts with `/platform` (the platform endpoints all share that prefix).

1. **`clearSession()` (lines 28-31)** — take the login target as a parameter (or read
   `window.location.pathname`). Redirect to `/platform/login` when in platform context,
   else `/login`. Keep `localStorage.removeItem('user')` for the tenant case only
   (platform store holds no `user` in localStorage — harmless but scope it).

2. **Refresh block (lines 40-73)** — do **not** attempt the tenant `/auth/refresh` for
   platform requests. Platform auth has no refresh flow here; a platform 401 is terminal.
   Simplest: gate the whole refresh branch on "not a platform request", so platform 401s
   fall straight through to step 3.

3. **Terminal redirect (lines 76-77)** — the `pathname !== '/login'` guard must become
   context-aware: skip the redirect when already on the matching login page
   (`/login` for tenant, `/platform/login` for platform). Otherwise `clearSession()`
   redirects to the correct login page for the context.

### Suggested signatures

```ts
// url-based: platform endpoints are the only ones under /platform
private isPlatformRequest(config?: { url?: string }): boolean {
  return !!config?.url?.startsWith('/platform')
}

private clearSession(platform: boolean): void {
  if (!platform) localStorage.removeItem('user')
  window.location.href = platform ? '/platform/login' : '/login'
}
```

Then in the interceptor compute `const platform = this.isPlatformRequest(config)` once and:
- guard the refresh branch with `!platform`
- change the `pathname !== '/login'` conditions to
  `window.location.pathname !== (platform ? '/platform/login' : '/login')`
- pass `platform` to `clearSession(platform)`

## Edge cases to cover

- **Already on the correct login page** → no redirect loop (both `/login` and
  `/platform/login`). Currently only `/login` is guarded; add `/platform/login`.
- **`getMe()` on the platform login page** (`platformAuth.service.ts:69`) returning 401
  must NOT redirect anywhere — caller already on `/platform/login`, so the guard above
  suppresses it. This is the direct repro of the reported bug.
- **Tenant flows unchanged**: tenant 401 → `/auth/refresh` → retry/clearSession → `/login`
  must behave exactly as today. Do not regress the refresh-queue logic (`isRefreshing`,
  `refreshQueue`).
- **`_retry` after a successful tenant refresh** that still 401s (role/authz) keeps the
  existing behavior (the `!config?._retry` guard at line 76 stays).

## Existing patterns to follow

- Auth-context split already exists: separate stores/services/routes for platform vs tenant
  (`stores/platform.store.ts`, `modules/platform/`, `router/index.ts:86-93`). The interceptor
  is the one place that ignored the split — this change aligns it.
- Keep the `// ponytail:` comment style already used at `api.service.ts:75`.

## Out of scope (note, don't fix)

- `main.ts` never calls `platformStore.initialize()`, so a hard refresh inside `/platform/*`
  drops the platform session and the guard bounces to `/platform/login`. That is *correct*
  post-fix behavior (goes to the right login now), not this bug. Leave it.

## Tests

Interceptor is plain logic — one small unit test on the redirect target:
- 401 on a `/platform/*` request while `pathname !== '/platform/login'` → target `/platform/login`, no `/auth/refresh` call.
- 401 on a tenant request → existing `/auth/refresh` path, target `/login`.
- 401 while already on the matching login page → no redirect.
Mock `window.location.href` assignment (jsdom) and the axios instance.

## Effort: medium — single-file, behavior-preserving for tenant, additive for platform.
