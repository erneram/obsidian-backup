# Changes — hospital-belen stage-2 (iteration 2)

## Files modified

### `hospital-belen-web/kairosaid/src/services/api.service.ts`

- Added `isSessionProbe(config?)` — returns true for `*/users/me` and `*/platform/me` endpoints.
- Added early-exit guard: if `status === 401 && isSessionProbe(config)` → `Promise.reject(error)` immediately, skipping both the refresh branch and `clearSession()`. Probe 401s are not mid-session failures; the router guard decides where to send the user.
- Iter-1 changes (`isPlatformRequest`, `clearSession(platform)`, `loginPage`) retained — still correct for non-probe mid-session platform 401s.

### `hospital-belen-web/kairosaid/src/main.ts`

- Imported `usePlatformStore`.
- Replaced unconditional `authStore.initialize()` with path-gated boot probe:
  - `/platform/*` → `usePlatformStore().initialize()` (probes `/platform/me`; restores platform session on reload)
  - everything else → `useAuthStore().initialize()` (tenant flow unchanged)

## What Tester should verify

1. **Apex `/platform/users`, no session** → resolves to `/platform/login`, NOT `/login`.
2. **Apex `/platform/users`, valid platform cookie** → platform session restored, page renders, no login bounce on reload.
3. **401 on `/users/me`** → no `/auth/refresh` call, no `window.location.href` write, promise rejects silently.
4. **401 on `/platform/me`** → same (no redirect).
5. **Mid-session 401 on a real platform API call** (e.g. `GET /platform/tenants`) → still redirects to `/platform/login` (iter-1 behavior).
6. **Tenant subdomain, expired session** → still lands on `/login` via router guard, unchanged.
7. **Tenant subdomain, valid session** → `authStore.initialize()` restores it, unchanged.
