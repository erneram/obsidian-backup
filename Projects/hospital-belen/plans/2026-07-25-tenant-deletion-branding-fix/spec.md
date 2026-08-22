# Spec — Platform roles/login fixes (hospital-belen)

Stage-1 · effort=medium · repo clean on `main` (both submodules). No pending branch diff; these are 4 new changes.

Reference UI: https://kairosaid.inksightdev.com/platform/roles

OPEN QUESTION: Task 4 "review each tenant for consistency" is a QA/verification step, not a code change — interpreted as a manual per-tenant checklist (see Verification). If a concrete code fix is expected, name the inconsistency.

---

## Task 1 — Tenant name/logo on the login face

**File:** `hospital-belen-web/kairosaid/src/modules/auth/pages/LoginPage.vue`

Currently the card header is hardcoded `Kairos Aid` (line 5). On a tenant subdomain the branding is already fetched in `main.ts` via `brandingService` and cached. Show the tenant's own name (and logo if present) instead of the hardcoded string.

- Read branding for the current host: `resolveTenantSlug()` + `brandingService.readCache(slug)` (type `TenantBranding` from `src/utils/theme.utils.ts`: `{ name, logoUrl }`).
- Render `branding.logoUrl` (if set) above the title, and `branding.name` as the `<h2>`. Fall back to `Kairos Aid` when no tenant / no cache (apex, localhost).
- Cache-only read is acceptable (branding is written to cache on every load). If a fresh first-visit name is required, expose a tiny reactive singleton (`ref<TenantBranding|null>`) set from `main.ts` after `brandingService.fetch`, and read it here. Prefer the reactive singleton — it's ~5 lines and avoids a stale/blank first paint.
- Do NOT fetch branding again in the component; reuse the existing service path.

Edge cases: logo missing → title only; tenant deleted → cache cleared in `main.ts`, falls back to default; keep the tenant-picker block unchanged.

---

## Task 2 — Modules list not persisting (platform roles → "Módulos" dialog)

**File:** `hospital-belen-web/kairosaid/src/modules/platform/pages/PlatformRolesPage.vue`

Backend verified correct: `PUT /api/platform/tenants/{tid}/roles/{rid}/menu-items` → `set_items_for_role` (DELETE-then-INSERT in a tx, `menu_item_role_repository.rs`). Route, DTO (`SetMenuItemsRequest.menu_item_ids`), and FK (group ids are real parent `menu_items`) are all fine. **The bug is frontend.**

**Root cause — standalone groups can't be toggled.** `/platform/modules` (`module.rs::get_modules`) returns two group shapes:
- `group_type: "group"` → has `modules[]` (children)
- `group_type: "standalone"` → `modules: []`, the group's own `id` is the real menu item

`moduleGroupState(group)` (line 568) only inspects `group.modules`; for a standalone (`mods=[]`, count 0) it always returns `false`, so the checkbox never reflects selection and the item appears to "not save". `toggleModule` (576) adds `group.id` but the UI still renders unchecked → user perceives no change persisted.

**Fix:**
- `moduleGroupState`: if `group_type === 'standalone'` (or `modules` empty), return `selectedModuleIds.has(group.id)`.
- `toggleModule`: for standalone, toggle only `group.id` (add/remove); keep current all-children logic for real groups.
- Verify grouped-module round-trip too: after Guardar, reopen must show the same checks (currently `saveModules` closes without reload — the reopen fetch is the real test).

Also confirm `selectedModuleIds` mutations trigger reactivity (Vue 3 tracks `Set` on a `ref`, so `.add/.delete` on `.value` is fine — no change needed unless observed otherwise).

**Manual repro:** platform admin → tenant → role → Módulos → toggle a standalone module (e.g. Dashboard) and a child module → Guardar → reopen → both must stay checked.

---

## Task 3 — Permissions modal: rows, not a single line

**File:** `PlatformRolesPage.vue`, permissions `<Dialog>` (lines 141–184)

Today permissions live inside per-module `Tabs`; within a tab each permission is one horizontal `<label>` row (`resource:action` chip + description). The complaint ("single line") is that a tab shows one cramped line-list and you can't scan across modules. Switch the per-module permission list from a flat vertical list to a **rows/grid layout** grouped by action so all permissions of a module are visible at once.

- Keep tabs OR replace with stacked module sections (see Diseño visual). Within a module, lay permissions out in a responsive grid: columns for `read / create / update / delete` (ACTION_ORDER already exists, line 433), one row per resource is not applicable here since a tab is already one resource — instead render the 4 actions as aligned cells so presence/absence reads at a glance.
- Preserve existing behavior: `moduleAdminState` header checkbox, `togglePermsModule`, `togglePerm`, the `n/total` counter.
- Keep the missing-action case graceful (not every module has all 4 actions → render an empty cell, not a broken row).

## Diseño visual

Admin CRUD surface inside the existing design system — restraint over novelty; match tokens, don't invent a palette.

- Use existing `@inksightdev/ui` primitives + Tailwind semantic tokens already in the file (`text-foreground`, `bg-muted/40`, `border-border`, the blue chip for `resource:action`). No new colors.
- **Permissions layout:** a 4-column grid per module — header row `Read · Create · Update · Delete`; each row a permission group with a Checkbox cell per action. Missing action → muted `—`. On mobile (`sm:` down) collapse to the current stacked rows. This turns the "single line" into a scannable matrix, the one deliberate improvement; everything else stays quiet.
- **Login face:** logo (max-h ~48px, centered) above the tenant name in the existing `text-3xl font-bold` treatment; subtitle unchanged. No layout risk — the branding IS the identity, let it show.

---

## Task 4 — Per-tenant consistency review (verification, likely no code)

Manual pass after tasks 1–3, for each tenant subdomain:
- Login face shows that tenant's name/logo (Task 1).
- Roles page: modules save + reopen round-trips (Task 2), for grouped and standalone modules.
- Permissions matrix renders (Task 3), counters correct.
- If a tenant is missing expected modules/roles → that's data, not code: report the tenant + gap, don't patch UI.

---

## Verification (tester)

- Frontend: `cd hospital-belen-web/kairosaid && npm run build` (typecheck) + `npm run lint`.
- Backend untouched — no API/Rust changes expected. If coder touches API, run `cargo test` + `cargo clippy` in `hospital-belen-api`.
- Behavioral: the two manual repros above (modules round-trip, permissions matrix, login branding via `?tenant=<slug>` dev override on localhost).

## Existing patterns to follow

- Branding read: `src/main.ts` (lines 8–31), `src/services/branding.service.ts`, `applyBranding` in `src/utils/theme.utils.ts`.
- Dialog + reactive Set toggles: the permissions block already in `PlatformRolesPage.vue` (`selectedPerms`, `togglePerm`) — mirror it for the grid.
- `// ponytail:` note already present at line 471; keep that convention for any deliberate shortcut.

No installable skill needed (`npx skills find rbac/permissions` → nothing beats the in-repo pattern).
