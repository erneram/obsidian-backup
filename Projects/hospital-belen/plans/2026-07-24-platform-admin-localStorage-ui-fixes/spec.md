# Spec — Bundle: platform session restore + 3 UI fixes

4 independent issues. 1 backend/store, 3 frontend. Files are disjoint — safe to do in any order.

---

## Issue 1 — Platform session not restored on hard reload

**Cause** (from prior audit): `platformStore.initialize()` (`stores/platform.store.ts:50-53`) only
awaits `/platform/me`; it has **no synchronous localStorage restore**. `main.ts:46-50` fires it
fire-and-forget, so on reload of a protected `/platform/*` route the router guard runs before
`/platform/me` resolves → `isAuthenticated=false` → bounce to `/platform/login`, even with a valid
cookie.

**Pattern to mirror**: the tenant side already does this correctly — `auth.store.ts:106-121`
(restore from localStorage synchronously, then verify via API) + `auth.service.ts:68-74,86-89`
(`getMe` caches `user`, `getCurrentUser` reads it).

**Files to modify**
- `src/modules/platform/services/platformAuth.service.ts`
- `src/stores/platform.store.ts`

**Changes**
1. `platformAuth.service.ts`:
   - Add a cache key, e.g. `const PLATFORM_USER_KEY = 'platformUser'`.
   - In `getMe()` (`:69-76`): on success `localStorage.setItem(PLATFORM_USER_KEY, JSON.stringify(user))`; keep returning `null` on failure (do NOT clear here — let the store decide).
   - In `logout()` (`:61-67`): `localStorage.removeItem(PLATFORM_USER_KEY)`.
   - Add `getCurrentUser(): PlatformUser | null` reading/parsing that key (guard `JSON.parse` in try/catch — mirror `auth.service.ts:86-89`).
2. `platform.store.ts`:
   - `login()` success path already sets `user.value` — also fine (getMe caches).
   - Rewrite `initialize()` to mirror `auth.store.ts:106-121`:
     ```ts
     async function initialize(): Promise<void> {
       const stored = platformAuthService.getCurrentUser()
       if (stored) user.value = stored          // synchronous — guard sees it immediately
       const me = await platformAuthService.getMe()
       if (me) user.value = me
       else { user.value = null; /* stale cache cleared by getMe? no — clear here */ }
     }
     ```
     On verify failure clear the cache too (`localStorage.removeItem(PLATFORM_USER_KEY)`), same as `auth.store.ts:119`.
   - `logout()` (`:45-48`) already sets `user.value = null`; the service now clears the cache.

**Edge cases**
- Stale/invalid cached user (cookie expired): synchronous restore shows the shell for one tick,
  then `/platform/me` 401 → `getMe` null → clear user + cache → guard/`PlatformLayout.onMounted:98-102`
  redirects to `/platform/login`. Acceptable (same behavior the tenant side has).
- Corrupt localStorage JSON → `getCurrentUser` returns null (try/catch), no crash.
- `PlatformLayout.onMounted` also calls `platformStore.initialize()` — now idempotent and cheap; fine.

---

## Issue 2 — Dashboard stat cards not theme-aware (light/dark)

**File**: `src/modules/dashboard/pages/DashboardPage.vue`

**Cause**: hardcoded non-theme colors on the stat grid.
- `:5` card container `bg-white` → stays white in dark mode.
- `:10` sub text `text-gray-400`, and `:174` low-stock idle color `text-gray-400` — hardcoded gray.

The tables below already use the correct tokens (`bg-card`, `border-border`, `text-muted-foreground`,
`bg-muted/50`) — follow that same convention.

**Changes**
- `:5` `bg-white` → `bg-card`.
- `:10` and `:174` `text-gray-400` → `text-muted-foreground`.
- The stat value accent colors (`text-blue-600`, `text-orange-600`, `text-green-600`,
  `text-emerald-700`, `text-yellow-600`, `text-red-600`) read acceptably on both themes — leave them,
  they are intentional per-metric accents (do NOT tokenize away the semantic colors). Only the
  surface (`bg-white`) and neutral gray text are the bug.

**Check**: toggle theme (user menu → Modo Oscuro); the 6 cards must sit on `bg-card`, not white.

---

## Issue 3 — Sidebar left-edge misalignment (header logo / module icons / footer avatar)

**File**: `src/layouts/MainLayout.vue` (+ the fact that `@inksightdev/ui` sidebar parts carry default
padding).

**Cause**: three different left insets stack up, so the header logo box, the module icons, and the
footer avatar each start at a different x:
- **Header** (`:14-24`): `SidebarHeader` default is `px-4` (lib) **plus** inner `<div class="… px-2">`
  → logo box left ≈ **24px**.
- **Module rows** (`:30,44`): `SidebarMenu class="px-2"` + `SidebarMenuButton` internal padding
  → icon left ≈ **16px**.
- **Footer** (`:97-98`): `SidebarFooter` default `p-4` (lib) + inner `<div class="… p-2">` + `Button
  class="… px-3">` → avatar left ≈ **36px**.

**Change — unify to one left inset (use the module row, ≈16px, as the reference)**
- Header: pass `px-2` on the `SidebarHeader` (tailwind-merge lets the passed class override the lib's
  `px-4`) and remove the inner `<div>`'s `px-2` so it isn't doubled. Goal: logo box left edge == module
  icon left edge.
- Footer: pass `p-2` on `SidebarFooter` (override lib `p-4`), and change the footer `Button` from
  `px-3` to `px-2` so the avatar's left edge lines up with the module icons above.
- Verify the collapsed-rail state (`sidebarCollapsed`) stays centered (`:102 justify-center`) after the
  padding change.

> Exact pixels are implementation detail — the coder must eyeball the three left edges against the
> rendered sidebar and land them on a single vertical line (both expanded and collapsed). The above
> classes are the recommended starting point, not sacred.

---

## Issue 4 — User email overflows nav, overlaps chevron

**File**: `src/layouts/MainLayout.vue:110-116`

**Cause**: `truncate` needs a width-constrained box. The name/email spans sit in
`<div class="flex flex-col items-start flex-1 min-w-0">`. `items-start` makes each child size to its
content (not stretch), so with `white-space:nowrap` the email span grows to full text width and
overflows past the `ChevronUp` (`:115`).

**Change**
- Add `w-full` (or `max-w-full`) to both truncating spans (`:112` name, `:113` email) so `truncate`
  has a bound: `class="text-xs text-muted-foreground truncate w-full"`. (`block`/`w-full` gives the
  truncate a definite width inside the `min-w-0` column.)
- Keep `min-w-0` on the column and `shrink-0` on the `ChevronUp` (`:115`, already present) so the chevron
  reserves its space and the text ellipsizes before reaching it.

**Check**: a long email (e.g. `administrador.general@hospital-belen.example.com`) ellipsizes and never
touches/overlaps the chevron, at both full and narrow sidebar widths.

---

## Diseño visual

No new design language — all four are corrective fixes to existing components. Direction is
**"match the design system already in the repo"**:
- **Theme**: use the established semantic tokens (`bg-card`, `bg-muted`, `border-border`,
  `text-foreground`, `text-muted-foreground`) that the dashboard tables and sidebar already use;
  never hardcode `bg-white` / `text-gray-*` for surfaces/neutrals. Per-metric accent colors stay.
- **Alignment**: one consistent left inset for the whole sidebar column (header brand, module icons,
  footer user) on an 8px grid — a single vertical line for all leading elements, expanded and collapsed.
- **Truncation**: text in fixed-width chrome always ellipsizes within its container; icons that share
  the row are `shrink-0`.

(frontend-design skill not invoked: no aesthetic exploration needed — the target look already exists
in-repo and these changes conform to it. ponytail.)

---

## Tests
- **Issue 1**: unit on `platform.store.initialize()` — with a cached user, `user.value` is set
  synchronously before the `getMe` await resolves; on `getMe` returning null the user + cache are
  cleared. Mock `platformAuthService`.
- **Issues 2-4**: visual/CSS — no unit test. Manual checklist in `test-results.md`: (2) dark-mode
  cards on `bg-card`; (3) three sidebar left edges aligned expanded+collapsed; (4) long email
  ellipsizes, no chevron overlap. Use `ce-test-browser` if the harness supports it.

## Effort: medium — 4 files, disjoint. Issue 1 is the only logic change; 2-4 are class edits.
