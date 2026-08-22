# Spec — CRITICAL rbac regression fixes (hospital-belen)

Stage-1 · effort=high · post-merge regressions from `87cf6f1 feat(rbac): role colors system + password endpoint`. All 3 root-caused. Fixes are small and surgical.

Repos: `hospital-belen-api` (Rust), `hospital-belen-web/kairosaid` (Vue). Migrations auto-run on boot via custom runner (`infrastructure/database/migrator.rs`, tracked in `schema_migrations`) — a new numbered `.sql` = "auto-run on push".

---

## Issue 1 — Module assignment 422  (root cause: serde camelCase on a snake_case payload)

**File:** `hospital-belen-api/src/web/dto/rbac.rs:69-73`
```rust
#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]      // ← THE BUG
pub struct SetMenuItemsRequest {
    pub menu_item_ids: Vec<Uuid>,
}
```
`rename_all = "camelCase"` makes the field deserialize from **`menuItemIds`**. `SetMenuItemsRequest` is shared by BOTH menu-items handlers:
- `rbac.rs:322` → `PUT /api/roles/{id}/menu-items` (tenant admin, `admin/pages/RoleListPage.vue` via `adminService.ts:79`, sends **`menuItemIds`** → works).
- `platform_rbac.rs:397` → `PUT /api/platform/tenants/{tid}/roles/{rid}/menu-items` (platform admin, `PlatformRolesPage.vue:663`, sends **`menu_item_ids`** → field missing → `Vec<Uuid>` can't deserialize → **422**).

The rename was added to match the legacy admin page and silently broke the newer platform page.

**Fix (align on snake_case — the convention everywhere else in the API: `role_id`, `new_password`, etc.):**
1. Remove the `#[serde(rename_all = "camelCase")]` line from `SetMenuItemsRequest`.
2. `hospital-belen-web/kairosaid/src/modules/admin/services/adminService.ts:78-79` — change the body to snake_case so the legacy tenant-admin page keeps working:
   ```ts
   async setRoleMenuItems(roleId: string, menuItemIds: string[]) {
     return apiService.put<void>(`/roles/${roleId}/menu-items`, { menu_item_ids: menuItemIds })
   }
   ```
`PlatformRolesPage.vue:663` already sends snake_case — leave it.

**Verify:** both role-module dialogs (`RoleListPage.vue` tenant admin + `PlatformRolesPage.vue` platform admin) save + reopen round-trip. Do NOT re-introduce the standalone-group bug (already fixed in `80b1795`).

---

## Issue 2 — Stale "Settings" / "Clinic" modules  (root cause: seeded menu_items never removed)

Seed rows (`migrations/002_seed.sql`):
- `section-settings` — label **"Configuración"** (top-level, sort 90) = the "Settings" module.
- `settings-clinic` — label **"Clínica"**, path `/settings/clinic`, child of `section-settings` = the "Clinic" module.

⚠️ Do **not** touch `section-clinical` (label "Clínica", sort 20) — that's the real clinical section with live children. Only the two `settings-*` names above.

**FK behavior (verified in `001_schema.sql`):**
- `menu_item_roles.menu_item_id → menu_items(id) ON DELETE CASCADE` → junction rows auto-clean.
- `menu_items.parent_id → menu_items(id) ON DELETE SET NULL` → deleting the parent does NOT delete the child, so delete the child by name too.

**Fix — new migration `hospital-belen-api/migrations/009_remove_settings_module.sql`:**
```sql
-- Remove deprecated Settings section and its Clínica child. Superseded by
-- per-tenant clinic settings; menu_item_roles rows cascade automatically.
DELETE FROM menu_items WHERE name IN ('settings-clinic', 'section-settings');
```
(No `005` exists — removed in `29415e5`; next free number is `009`, after `008_role_color.sql`.) Runs on next backend boot. `get_modules` already drops childless non-standalone groups, so no code change needed. Idempotent (DELETE of absent rows is a no-op) — safe on re-run and on fresh DBs.

---

## Issue 3 — `GET /api/users/me/roles` 403  (root cause: /me sub-routes collide with dynamic endpoint regex)

Auth is DB-driven (`middleware/dynamic_auth.rs`): every request is matched against `endpoints` rows (method + `path_regex`), sorted by specificity (fewer `[^/]+` wildcards first — `reload()` lines 78-83). A matched row's `required_permissions` are enforced.

The new self-routes `/api/users/me/roles`, `/me/permissions`, `/me/password` (added in `87cf6f1`, `router.rs:122-124`) have **no `endpoints` rows**. So they fall through to the existing broader patterns:
- `^/api/users/[^/]+/roles$` (`002_seed.sql:697`, "List user roles") → **matches `/api/users/me/roles`** and carries the user-roles permission (linked via `002_seed.sql:885` `LIKE '/api/users/%/roles%'`) → **403** for a user without it.
- Same collision for `/me/permissions` and `PUT /me/password` (→ `/api/users/{id}/password`, needs `user:update`).

`/api/users/me` itself works only because it has an explicit 0-wildcard row (`002_seed.sql:690`) that sorts ahead of `^/api/users/[^/]+$`. The sub-routes need the same treatment.

**Fix — add explicit 0-wildcard endpoint rows (append to migration `009`, or a sibling `010_seed_me_endpoints.sql`):**
```sql
-- Self-service /me sub-routes: authenticated, no special permission. Zero-wildcard
-- regexes sort ahead of /api/users/{id}/... so they match first (dynamic_auth
-- specificity order) and expose no required_permissions → auth-only pass-through.
INSERT INTO endpoints (method, path, path_regex, description, is_public, is_active) VALUES
  ('GET', '/api/users/me/roles',       '^/api/users/me/roles$',       'Get own roles',       false, true),
  ('GET', '/api/users/me/permissions', '^/api/users/me/permissions$', 'Get own permissions', false, true),
  ('PUT', '/api/users/me/password',    '^/api/users/me/password$',    'Change own password', false, true)
ON CONFLICT DO NOTHING;
```
- Match the `endpoints` column list + `ON CONFLICT` target used by `002_seed.sql` (confirm the unique key — likely `(method, path)` — and mirror it). Attach **no** permission rows.
- Order-safe on fresh installs: `002_seed`'s `LIKE '/api/users/%/roles%'` permission link runs before `009/010`, so these rows never receive permissions.
- The `EndpointCache` reloads from DB on startup (`reload_from_repo`), so a redeploy picks the rows up. **If the app also exposes a runtime endpoint-cache reload, no restart needed; otherwise the boot reload covers it** — confirm the cache is reloaded after migrations run.

**Verify:** as a plain tenant user (no `user:read`/`user:update`) on the `hospital-belen` subdomain, `GET /api/users/me/roles` and `/me/permissions` return 200; profile loads name + role; `PUT /api/users/me/password` (self password change) is not 403.

---

## Root-cause summary
| Issue | Root cause | Fix | Files |
|---|---|---|---|
| 1 · 422 | `#[serde(rename_all="camelCase")]` on shared `SetMenuItemsRequest` vs snake_case payload | drop rename + snake_case the legacy caller | `dto/rbac.rs`, `adminService.ts` |
| 2 · stale modules | seeded `section-settings`/`settings-clinic` never removed | DELETE migration (CASCADE cleans junction) | `migrations/009_*.sql` |
| 3 · 403 | `/me/*` sub-routes match `/users/{id}/*` regex, inherit its permissions | seed 0-wildcard endpoint rows, no perms | `migrations/009|010_*.sql` |

## Verification (tester)
- Backend: `cd hospital-belen-api && cargo test && cargo clippy`; apply migrations to a scratch DB, confirm `section-settings`/`settings-clinic` gone and the 3 `/me` endpoint rows present.
- Frontend: `cd hospital-belen-web/kairosaid && npm run build && npm run lint`.
- E2E (all three): platform admin assigns modules to a role (no 422, persists); Settings/Clinic absent from module lists; non-admin tenant user's profile loads (no 403 on `/me/roles`).

## Existing patterns to follow
- Endpoint seed row shape: `002_seed.sql:690` (`/api/users/me`).
- Migration file style + custom runner: `008_role_color.sql`, `migrator.rs`.
- Dynamic authz + specificity sort: `middleware/dynamic_auth.rs:60-106`.

No installable skill applies (`npx skills find "axum rbac endpoint regex"` → nothing beats in-repo patterns). No UI/design work this cycle.
