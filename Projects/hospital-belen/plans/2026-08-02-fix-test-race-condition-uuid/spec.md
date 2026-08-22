# Spec — Modules i18n + sidebar/admissions UI cleanup

Branch (both repos): `fenix/modules-i18n-admissions-cleanup` — new work, NOT the seeds PRs.
Frontend = `hospital-belen-web/kairosaid`. Backend = `hospital-belen-api`.

Modules architecture (read first): sidebar is data-driven. `GET /api/modules`
(`hospital-belen-api/src/web/handlers/module.rs`) returns groups built from the
`menu_items` table. Frontend renders them in `layouts/MainLayout.vue` via
`module.store` / `module.service.ts`. Each item carries a stable `key` (= menu_item
`name`) and a Spanish `label` string.

---

## Task 1 — Inventory sidebar: wrong item highlighted (frontend)

**Root cause (confirmed):** inventory children paths (migrations/002_seed.sql:503-507):
`inventory-items` → `/inventory`, `inventory-products` → `/inventory/products`,
`inventory-movements` → `/inventory/movements`, `warehouses` → `/inventory/warehouses`.
`isRouteActive` (MainLayout.vue:298) is `route.path === path || route.path.startsWith(path + '/')`.
On `/inventory/products` the `/inventory` item ("Ítems") ALSO matches via prefix, so two
items highlight. That's the "selected state showing incorrectly (bg highlight)".

**Fix (MainLayout.vue):** most-specific match wins. A module/standalone is active only if
its path is the *longest* path (among all module + standalone paths) that matches the
current route (exact or prefix). Concretely:
- Collect all candidate paths (standalone `group.path` + every `group.modules[].path`).
- `bestMatch(route.path)` = the longest candidate `p` where `route.path === p || route.path.startsWith(p + '/')`.
- `isModuleActive(path)` / `isStandaloneActive` → `path === bestMatch`.
- `isGroupActive(group)` → `group.modules.some(m => m.path === bestMatch)`.

This keeps detail routes working (`/patients/123` → "Pacientes" `/patients`) while
`/inventory/products` no longer lights up `/inventory`.

Edge cases to cover:
- On exactly `/inventory` → "Ítems" active, siblings not.
- On `/inventory/products` → only "Productos" active; parent group still shows active.
- On `/patients/123` (detail) → "Pacientes" still active (longest match is `/patients`).

## Task 5 — Modules: i18n-compatible labels (frontend)

Today MainLayout renders raw backend Spanish: `{{ group.label }}` (lines 35, 48) and
`{{ mod.label }}` (line 81). Backend already returns a stable `key` per item — no backend
change needed.

**Refactor:**
- Add a `modules` namespace to `locales/es.json` keyed by menu_item `name`:
  `dashboard`, `section-clinical`, `patients-list`, `medical-records`, `prescriptions`,
  `section-scheduling`, `appointments-calendar`, `appointments-list`, `section-admin`,
  `admin-tenants`, `users-list`, `roles-list`, `endpoints`, `section-hr`, `employees-list`,
  `schedules`, `leave-requests`, `pediatrics`, `pediatrics-growth-charts`, `section-billing`,
  `receipts`, `account-stmts`, `estado-cuenta-new`, `packages`, `section-inventory`,
  `inventory-products`, `inventory-items`, `inventory-movements`, `warehouses`,
  `section-reports`, `report-appointments`, `report-billing`, `report-inventory`,
  `section-settings` (+ any settings children). Values = current Spanish labels.
  (`locales/es.json` is the only locale file today; there is no en.json — create the
  `modules` keys in es.json; adding en.json is optional/out of scope unless the human wants it.)
- In MainLayout, add a helper using vue-i18n `te`/`t` (pattern: `plugins/i18n.ts`):
  `moduleLabel(key, fallback) = te('modules.' + key) ? t('modules.' + key) : fallback`.
  Replace the three `.label` interpolations with `moduleLabel(group.key, group.label)` /
  `moduleLabel(mod.key, mod.label)`. Fallback to backend `label` so untranslated keys never
  render blank.

Follow the existing `$t(...)` usage already in `AdmissionListPage.vue` (`admissions.*` keys)
for consistency.

## Task 4 — Modules: remove duplicate admissions under Clínica (backend migration)

`/admissions` (component `AdmissionListPage`) is seeded twice (migrations/002_seed.sql):
- `admissions` "Ingresos / Altas" `/admissions` under `section-clinical` (line 435).
- `account-stmts` "Estados de Cuenta" `/admissions` under `section-billing` (line 492).

Keep the Facturación one (`account-stmts`); remove the clinic one (`admissions`).

**Fix — new migration** `hospital-belen-api/migrations/012_remove_duplicate_admissions_menu.sql`
(next free number; 011 is the highest). Idempotent:
```sql
-- Remove role grants then the menu item (name is unique).
DELETE FROM menu_item_roles
 WHERE menu_item_id IN (SELECT id FROM menu_items WHERE name = 'admissions');
DELETE FROM menu_items WHERE name = 'admissions';
```
Check the actual FK on `menu_item_roles.menu_item_id` first — if it's `ON DELETE CASCADE`
the first DELETE is redundant; keep it anyway for safety on envs without cascade.
Also delete the `admissions` row from the 002 CROSS JOIN (line 435) so fresh DBs don't
recreate it — but the migration is what fixes existing tenants.
Do NOT touch the `admissions` permission rows (002_seed.sql:54-56) — those are RBAC
permissions, unrelated to the menu item.

Verify: `GET /api/modules` no longer lists an item under "Clínica" pointing at `/admissions`;
"Estados de Cuenta" under "Facturación" remains.

## Task 2 — Admissions detail: remove "Aplicar paquete" button (frontend)

`modules/admissions/pages/AdmissionDetailPage.vue`, lines 110-114 — remove the button:
```
<Button v-if="detail.statement.status !== 'CLOSED'" size="sm" variant="outline" @click="openApplyPackage">
  <Plus :size="14" class="mr-1" /> Aplicar paquete
</Button>
```
Keep "Editar" (`openEditPackage`) and "Agregar cargo". After removing, check whether
`openApplyPackage` and the "Aplicar Paquete Médico" dialog (line ~496) plus its state
(`selectedPkgId`, `pkgQuery`, `availablePackages`, apply handler) become unused — if so,
delete the dead code too (ponytail). If "Editar" reuses that dialog, leave the dialog.

**OPEN QUESTION:** with the button gone, "Editar" only renders when a package/charge already
exists (its `v-if` requires `packageItems.length || inventoryChargeItems.length`). How does a
user apply the FIRST package? Options: (a) make "Editar" always visible and handle the empty
case, (b) show "Aplicar paquete" only while no package is applied yet. Human to confirm intent
of "one package at a time only" before the Coder deletes the entry point outright.

## Task 3 — Admissions list: remove "No Admisión" column (frontend)

`modules/admissions/pages/AdmissionListPage.vue`:
- Remove header `<TableHead>{{ $t('admissions.admissionNumber') }}</TableHead>` (line 47).
- Remove body cell `<TableCell ...>{{ row.admission.admissionNumber }}</TableCell>` (line 61).
- Decrement the empty-row `colspan="7"` → `6` (line 57).
No other column depends on it. Leave the `admissions.admissionNumber` i18n key in place
(harmless) unless unused elsewhere.

---

## Files to change
Frontend (`hospital-belen-web/kairosaid`):
- `src/layouts/MainLayout.vue` (tasks 1 + 5)
- `src/locales/es.json` (task 5 — add `modules` namespace)
- `src/modules/admissions/pages/AdmissionDetailPage.vue` (task 2)
- `src/modules/admissions/pages/AdmissionListPage.vue` (task 3)

Backend (`hospital-belen-api`):
- `migrations/012_remove_duplicate_admissions_menu.sql` (new, task 4)
- `migrations/002_seed.sql` (task 4 — drop `admissions` from clinical CROSS JOIN, fresh-DB only)

## Diseño visual (task 1 only)
Sidebar active state: exactly ONE leaf item highlighted per route (the most specific).
Parent group keeps its existing active affordance when a child is active. No new colors or
spacing — reuse the current `bg-accent/60 text-accent-foreground` active classes
(MainLayout.vue:70). frontend-design skill not invoked: no new UI, only correcting which
element receives the existing active style.

## Notes
- Commit frontend + backend changes to `fenix/modules-i18n-admissions-cleanup` in their repos.
- Backend touches migrations → run `cargo clippy --all-targets -- -D warnings` and a fresh-DB
  boot before pushing (see memory `hospital-belen-ci-clippy-gate`,
  `hospital-belen-fresh-db-admin-seed`).
- SKILL_RECOMENDADA: none.
