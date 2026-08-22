# Changes — Fix: Platform Admin toasts never render (missing Sonner host)

## Root cause confirmed

`<Sonner>` was mounted only inside `RouterLayoutView.vue` (the tenant app shell). Platform Admin routes are spread at the router top level under `PlatformLayout.vue`, which does NOT render `RouterLayoutView`. So every `toast.success/error` call in the platform section — and the interceptor's error toasts — executed with no renderer and produced nothing visible.

## Fix (2 files, 4-line net diff)

### `src/App.vue`
- Added `import { Sonner } from '@inksightdev/ui'` and `import { useTheme } from '@/composables/useTheme'`.
- Added `const { theme } = useTheme()`.
- Added `<Sonner position="top-right" richColors :closeButton="true" :duration="4000" :theme="theme" />` to template — same props as the removed instance.
- Result: one global host covers all route trees (tenant, platform, auth, public).

### `src/layouts/RouterLayoutView.vue`
- Removed `<Sonner …>`, its `Sonner` import, `useTheme` import, and `theme` binding.
- Result: no duplicate host; tenant app still uses the global one in App.vue.

## Verification

- `eslint src/App.vue src/layouts/RouterLayoutView.vue` → 0 errors ✓
- `prettier --check` on both files → "All matched files use Prettier code style!" ✓

## Repos tocados

- **web** (`hospital-belen-web`): branch `fenix/fix-toast-host`, 1 commit, pushed.

## Tester must verify (runtime — tests cannot cover this)

1. Log into Platform Admin → create/edit/delete a role or override → **success toast appears** top-right.
2. Force a 500 on a platform request → **error toast appears** (interceptor now has a host).
3. Tenant app mutation → exactly **one** toast, no duplicates (no double host).
4. Dark/light theme toggle → toast theme follows correctly (useTheme reactive in App.vue).
5. `npm run lint` + `npm run format:check` still green.
