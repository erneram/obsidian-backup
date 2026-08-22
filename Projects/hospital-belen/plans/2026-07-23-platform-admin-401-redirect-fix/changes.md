# Changes — hospital-belen stage-2

## Files modified

### `hospital-belen-web/kairosaid/src/services/api.service.ts`

- Added `isPlatformRequest(config?)` — returns true when `config.url` starts with `/platform`.
- Updated `clearSession(platform: boolean)` — redirects to `/platform/login` (platform) or `/login` (tenant); only removes `user` from localStorage for tenant.
- Interceptor now computes `platform` and `loginPage` once per error.
- Refresh block (`/auth/refresh` + queue) gated on `!platform` — platform 401s are terminal, no refresh attempt.
- Both `pathname !== '/login'` guards replaced with `pathname !== loginPage` — covers `/platform/login` too.

## What Tester should verify

1. Platform 401 (e.g. `GET /platform/me` while unauthenticated) → redirect to `/platform/login`, no `/auth/refresh` call.
2. Tenant 401 → `/auth/refresh` attempted, then retry; on refresh failure → redirect to `/login`.
3. 401 while already on `/platform/login` → no redirect (suppress loop from `getMe()` on platform login page).
4. 401 while already on `/login` → no redirect (existing behavior preserved).
5. `localStorage.removeItem('user')` NOT called for platform 401s.
