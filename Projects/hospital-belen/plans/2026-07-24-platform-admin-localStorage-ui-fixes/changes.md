# Changes — hospital-belen stage-2 (iteration 3, bundle)

## Files modified

### `src/modules/platform/services/platformAuth.service.ts`
- Added `PLATFORM_USER_KEY = 'platformUser'` constant.
- `getMe()`: caches user to `localStorage` on success.
- `logout()`: clears `localStorage` key.
- Added `getCurrentUser()`: synchronous localStorage read with try/catch, returns `PlatformUser | null`.

### `src/stores/platform.store.ts`
- Rewrote `initialize()`: synchronously restores from `getCurrentUser()` first (so router guard sees `isAuthenticated=true` immediately), then awaits `getMe()`; on null, clears user and cache.

### `src/modules/dashboard/pages/DashboardPage.vue`
- Line 5: `bg-white` → `bg-card` (stat card surface now theme-aware).
- Line 10: `text-gray-400` → `text-muted-foreground` (sub text token).
- Line 174: low-stock idle color `text-gray-400` → `text-muted-foreground`.

### `src/layouts/MainLayout.vue`
- Sidebar header: `px-2` on `SidebarHeader` (overrides lib `px-4`); removed inner div `px-2` to avoid doubling.
- Sidebar footer: `p-2` on `SidebarFooter` (overrides lib `p-4`); removed inner div `p-2`; button `px-3` → `px-2`.
- Email/name spans: added `w-full` to both truncating spans so `truncate` has a definite width within the `min-w-0` column.

## What Tester should verify

1. **Issue 1 — platform session restore**: reload `/platform/users` with a valid platform cookie → page renders, no bounce to `/platform/login`. Reload without cookie → bounces to `/platform/login` (not `/login`).
2. **Issue 2 — dark mode**: toggle Modo Oscuro; 6 stat cards sit on `bg-card`, not white. Low-stock idle value uses muted color.
3. **Issue 3 — sidebar alignment**: header logo, module icons, and footer avatar left edges align on a single vertical line (expanded and collapsed).
4. **Issue 4 — email truncation**: long email (`administrador.general@hospital-belen.example.com`) ellipsizes; chevron not overlapped at any sidebar width.
