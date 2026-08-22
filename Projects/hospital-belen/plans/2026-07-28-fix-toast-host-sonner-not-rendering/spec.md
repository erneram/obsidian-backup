# Spec — Fix: Platform Admin success/error toasts never render

**Project:** hospital-belen · **effort:** high · **stage-1 (investigation → remediation)**
**Scope:** `hospital-belen-web/kairosaid`.

## ROOT CAUSE (definitive)

The `<Sonner>` toast **host** is mounted in exactly one place —
`src/layouts/RouterLayoutView.vue` (the tenant app shell) — and **nowhere else** in the
app. Platform Admin routes do **not** render under that layout, so no toast host exists
in the Platform Admin tree. Every `toast.success()` / `toast.error()` call in the
platform section (and the interceptor's error toasts for `/platform/*`) executes
successfully but has **no renderer**, so nothing appears.

Evidence:
- `<Sonner>` occurs only in `RouterLayoutView.vue` (`grep` app-wide: RouterLayoutView=1,
  App.vue=0, PlatformLayout.vue=0).
- `src/router/index.ts:36` — tenant routes are children of `RouterLayoutView.vue` (has
  `<Sonner>`). `src/router/index.ts:71` — `...platformRoutes` are spread at top level,
  under `PlatformLayout.vue` (from `src/modules/platform/router.ts`), which has no
  `<Sonner>`.
- `App.vue` (the actual mounted root, `main.ts:42`) renders only `<router-view/>` — no
  `<Sonner>`. So auth/public layouts have no host either.
- History: `PlatformLayout.vue` was introduced (`245c806`) without a toast host; the
  toast work added `toast.*` calls but never a host in the platform tree. This is a
  pre-existing structural gap, **not** a regression from the toast/interceptor/lint/
  prettier deployment.

## Investigation checklist — findings (all 5 points)

1. **Are success toasts called?** ✅ Yes. 32 `toast.success/error` call sites across
   platform pages; guarded correctly (e.g. `PlatformRolesPage.vue`, `PlatformTenantDetailPage.removeOverride` after the fix). Calls do execute.
2. **Imports correct (`@inksightdev/ui`)?** ✅ Yes. All 12 platform files import
   `{ toast }` (or `toast` in a group) from `@inksightdev/ui`; `Sonner` also from there.
   Not the cause.
3. **Interceptor success handling broken by prettier/eslint?** ❌ No. Success path is
   `response => response` (`api.service.ts:44`), untouched; prettier reflow of
   `api.service.ts` changed only wrapping. The `<span>` prettier-ignore fix
   (`90859ed`) is template-only, unrelated. Not the cause.
4. **`apiService.raw` shared (single instance)?** ✅ Yes. `get raw()` returns the one
   `this.api`; all 5 platform services do `const http = apiService.raw`. No duplicate
   axios instances. Not the cause.
5. **Build/bundling issue?** ❌ No. `toast` is imported and used, so not tree-shaken;
   imports resolve. The failure is a missing **runtime host**, not a bundling fault.

**Conclusion:** points 1–5 are all healthy. The single defect is the missing
`<Sonner>` host in the Platform Admin (and auth/public) route tree.

## REMEDIATION

Mount the toast host **once at the true app root** so it covers every route (tenant,
platform, auth, public) instead of only the tenant shell.

### Files

**`src/App.vue`** — add the global `<Sonner>` here (this is the mounted root wrapping
all routes via `<router-view/>`):
```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useAuthStore } from '@/stores/auth.store'
import { Sonner } from '@inksightdev/ui'
import { useTheme } from '@/composables/useTheme'

const authStore = useAuthStore()
const { theme } = useTheme()

onMounted(() => {
  authStore.initialize()
})
</script>

<template>
  <router-view />
  <Sonner position="top-right" richColors :closeButton="true" :duration="4000" :theme="theme" />
</template>
```

**`src/layouts/RouterLayoutView.vue`** — **remove** the now-duplicate `<Sonner>` (and
its `useTheme`/`Sonner` imports) so the tenant section doesn't render two hosts:
```vue
<template>
  <MainLayout>
    <router-view />
  </MainLayout>
</template>

<script setup lang="ts">
import MainLayout from '@/layouts/MainLayout.vue'
</script>
```

Copy the exact `<Sonner>` prop set from `RouterLayoutView.vue` (`position="top-right"`,
`richColors`, `:closeButton="true"`, `:duration="4000"`, `:theme="theme"`) — don't
change toast styling/behavior for the tenant app.

### Why this is the right fix (not the alternative)

- One global host = toasts work in **all** sections (platform, tenant, auth, public
  booking) with a single config. The interceptor's `/platform/*` error toasts also
  start rendering.
- Rejected alternative: add a second `<Sonner>` to `PlatformLayout.vue` — duplicates
  config, still leaves auth/public pages hostless, and risks double hosts if a route
  ever nests both.

## Edge cases

- **No double host during transition.** Ensure `<Sonner>` is removed from
  `RouterLayoutView.vue` in the same change; two hosts can double-render or fight over
  the theme.
- **Theme sync.** `useTheme()` must live where `<Sonner>` is (now `App.vue`). Verify
  dark/light still tracks (there's prior art: commit `70a7c62` "Sonner theme sync").
- **SSR/initial mount.** `App.vue` mounts before any route resolves, so the host is
  present before the first `toast.*` call — good.

## Existing patterns to follow

- `<Sonner>` usage + theme binding: current `src/layouts/RouterLayoutView.vue` (the
  block being moved).
- Per-page success toast on mutation: `src/modules/platform/pages/PlatformRolesPage.vue`.

## Verification (must actually observe a toast in the platform section)

1. Run the app, log into Platform Admin, perform a create/edit/delete (e.g. create a
   role, delete an override) → **success toast appears** top-right.
2. Force a platform request error (500/403) → **error toast appears** (interceptor host
   now present).
3. Regression: tenant app mutation → still exactly one toast (no duplicates).
4. `npm run lint` + `npm run format:check` still green.

## Out of scope

- No changes to toast call sites (they are correct).
- No interceptor logic changes.
- No new dependency.
