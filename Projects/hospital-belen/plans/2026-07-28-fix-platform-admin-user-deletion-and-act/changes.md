# Changes — fenix/fix-platform-user-delete-500

## Files changed

### hospital-belen-api

- `src/infrastructure/database/base.rs`
  - Extended `DeleteMode::SoftSetField` with `cast: Option<&'static str>` field.
  - `_soft_delete_set_field`: builds `Expr::val(value).cast_as(Alias::new(ty))` when `cast` is `Some`, plain `value.into()` when `None`.
  - Updated `delete` match arm to destructure and forward `cast`.

- `src/infrastructure/database/repositories/user_repository.rs`
  - Added `cast: Some("user_status")` to `UserBmc::DELETE_MODE`.

### hospital-belen-web

- `kairosaid/src/modules/platform/pages/PlatformTenantDetailPage.vue`
  - Danger zone: replaced single deactivate button with `v-if/v-else` pair — deactivate shown when `tenant.isActive`, activate shown when inactive.
  - Added `activating` ref and `handleActivate` handler (mirrors `handleDeactivate`, calls `platformTenantService.update(tenantId, { isActive: true })`).

- `kairosaid/src/locales/es.json`
  - Added `"activate": "Activar tenant"` next to `"deactivate"`.

## Repos tocados

- `api` (hospital-belen-api)
- `web` (hospital-belen-web)

## What the Tester should review

- `DELETE /api/platform/tenants/{tenant}/users/{user}` as platform admin → 204, `users.status` = INACTIVE.
- Deleting already-INACTIVE user → still 204 (idempotent UPDATE).
- Deleting non-existent user → 404.
- Tenant detail danger zone: inactive tenant shows "Activar tenant" (default variant); clicking it re-activates and flips button back to destructive deactivate.
- Active tenant shows only deactivate button (no regression).
