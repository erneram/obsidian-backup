# Changes — modules-i18n-admissions-cleanup

## hospital-belen-web

- `src/layouts/MainLayout.vue`
  - T1: replaced `isRouteActive` prefix logic with `bestMatch()` (longest-match wins); `isModuleActive`, `isStandaloneActive`, `isGroupActive` all delegate to it.
  - T5: added `useI18n` + `moduleLabel(key, fallback)` helper; replaced 3 `.label` interpolations in template.
- `src/locales/es.json` — T5: added `modules` namespace with 35 keys (all menu_item names → current Spanish labels).
- `src/modules/admissions/pages/AdmissionDetailPage.vue` — T2: "Aplicar paquete" button now `v-if` includes `!packageItems.length && !inventoryChargeItems.length`; dialog/state untouched.
- `src/modules/admissions/pages/AdmissionListPage.vue` — T3: removed "N° Admisión" `<TableHead>` and `<TableCell>`, decremented empty-row colspan 7→6.

## hospital-belen-api

- `migrations/012_remove_duplicate_admissions_menu.sql` — T4: idempotent DELETE of `admissions` menu item (and its role grants). CASCADE FK handles grants; explicit DELETE kept for safety.
- `migrations/002_seed.sql` — T4: removed `admissions` row from clinical CROSS JOIN so fresh DBs don't recreate the duplicate.

## CI / test-infra fixes (rounds 2–5, also cherry-picked to fenix/seeds-ui-and-cross-tenant-packages)

- `.github/workflows/ci-backend.yml` — `--no-fail-fast` added; all failures surface in one run
- `Cargo.toml` — `serial_test = "3"` dev-dependency added
- `tests/common/mod.rs` — `provision_tenant` + `provision_user_in_tenant` helpers added
- `tests/seed_sharing.rs` — UUID-suffix all snapshot names; `recapture` test provisions own tenants + asserts 201 before reading id
- `tests/user_soft_delete_filtering.rs` — mutating tests provision own tenant + users; `#[serial]`
- `tests/platform_tenant_user_delete.rs` — mutating tests provision own tenant + users; `#[serial]`; unauth test uses random UUIDs
- `tests/tenant_deletion.rs` — all deleting tests provision own tenant; `#[serial]`; unauth test uses random UUID
- `tests/admission_update.rs` — `#[serial]` on mutating tests

## Repos tocados

- `web` — 1 commit (4 files)
- `api` — multiple commits (CI config, test infra, 2 migration files)

## Tester notes

- T1 edge cases: `/inventory` → only "Ítems" active; `/inventory/products` → only "Productos" active; `/patients/123` → "Pacientes" active.
- T2: "Aplicar paquete" hidden when `packageItems.length > 0` OR `inventoryChargeItems.length > 0`; "Editar" still shows when items exist.
- T3: admissions table renders 6 columns; no functional regression.
- T4: `GET /api/modules` must not list an item under "Clínica" pointing to `/admissions`; "Estados de Cuenta" under "Facturación" must remain. Requires fresh-DB boot (migration runs on startup).
- clippy: passed clean (`cargo clippy --all-targets -- -D warnings`).
- vue-tsc: passed clean.
