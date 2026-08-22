# Changes — hospital-belen — stage-2 (DISPATCH 3)

## Backend

### `migrations/008_role_color.sql` (NEW)
- Adds `color varchar(7)` column to `roles` table with hex CHECK constraint.

### `src/infrastructure/database/idens.rs`
- Added `Color` variant to `RoleIden` enum.

### `src/domain/rbac/role.rs`
- Added `pub color: Option<String>` to `Role`, `RoleForCreate`, `RoleForUpdate`.

### `src/infrastructure/database/repositories/role_repository.rs`
- Added `RoleIden::Color` to explicit column lists in `list()` and `find_by_slug()`.

### `src/web/dto/rbac.rs`
- Added `is_valid_hex_color()` validator helper.
- Added `color: Option<String>` to `CreateRoleRequest` and `UpdateRoleRequest`.

### `src/web/handlers/platform_rbac.rs` + `src/web/handlers/rbac.rs`
- `create_role` / `create_role_in_tenant`: hex color validation + `color` field forwarded to `RoleForCreate`.
- `update_role` / `update_role_in_tenant`: same for `RoleForUpdate`.

### `src/domain/user/mod.rs` + `src/web/dto/user.rs`
- Added `created_at: OffsetDateTime` to `User` struct and `UserResponse`.

### `src/web/handlers/user.rs`
- `get_me_roles`: new handler — returns full role objects (with color) for current user.
- `get_me_permissions`: new handler — returns union of permissions across all user roles (deduplicated by UUID).

### `src/web/router.rs`
- Routes added: `GET /api/users/me/roles`, `GET /api/users/me/permissions`.

## Frontend

### `src/utils/roleColor.ts` (NEW)
- `roleColor(role)`: returns `#7C3AED` for superadmin, explicit color if set, else slug-hash palette fallback.
- `roleBadgeStyle(color)`: tinted badge inline style (15% opacity background).

### `src/modules/platform/types/admin.ts`
- Added `color?: string | null` to `Role` interface.

### `src/modules/platform/services/platformAdmin.service.ts`
- Added `color?: string | null` to `createRole` payload type.

### `src/modules/platform/pages/PlatformRolesPage.vue`
- **Standalone module group bug fix**: `moduleGroupState` now handles groups with empty `modules[]` by checking `selectedModuleIds.has(group.id)` directly.
- **Permissions modal**: replaced flat permission list with 4-column grid (Read/Create/Update/Delete) using `findPerm()` helper; responsive (`grid-cols-1 sm:grid-cols-4`).
- **Superadmin-first sort**: roles sorted with superadmin first, then alphabetical by name.
- **Color dot**: color dot badge in role table rows (non-superadmin roles).
- **Color picker**: native `<input type="color">` in create and edit dialogs; "sin color" clear button; "+ agregar color" when null.

### `src/modules/platform/components/TenantAccordion.vue`
- Color dot before role name in user roles dialog.

### `src/composables/useBranding.ts` (NEW)
- Singleton `activeBranding` ref shared between `main.ts` and `LoginPage.vue`.

### `src/main.ts`
- Sets `activeBranding.value` on cache hit, network fetch, and null (deleted tenant reset).

### `src/modules/auth/pages/LoginPage.vue`
- Shows tenant logo (`<img>`) and name from `activeBranding`; falls back to "Kairos Aid".

### `src/modules/profile/types/index.ts`
- Removed `UserRole` dependency; new fields: `roleSlug`, `roleName`, `roleColor`; added `Permission` interface.

### `src/modules/profile/services/profileService.ts`
- Full rewrite: calls `GET /users/me` + `GET /users/me/roles` for profile; `GET /users/me/permissions` for permissions.

### `src/modules/profile/composables/useProfile.ts`
- Simplified: loads profile + permissions in parallel; exposes `profile`, `effectiveAllowed`, `effectiveDenied`.

### `src/modules/profile/pages/ProfilePage.vue`
- `roleLabel` uses `profile.roleName`; colored role badge via `roleBadgeStyle`.
- Removed obsolete roleAllowed/roleDenied/userAllowed/userDenied PermissionBuckets (now a single effective bucket).

---

## What the Tester should specifically review

1. **Migration 008**: verify migration runs without error on a fresh DB; `roles` table has `color` column.
2. **Role create/update API**: `color: "#1A2B3C"` is accepted; `color: "invalid"` returns 422; omitting `color` leaves it unchanged.
3. **Role colors UI**: color dot appears in PlatformRolesPage table; color picker opens in create/edit dialogs; "sin color" clears it; superadmin row has no color picker.
4. **Standalone module bug**: a module group with no sub-modules correctly toggles between checked/unchecked (not stuck at false).
5. **Permissions modal layout**: 4-column desktop grid (Read/Create/Update/Delete); collapses to stacked on mobile.
6. **Login branding**: logo and tenant name appear on `/login` for tenants with branding; fallback "Kairos Aid" for deleted/unknown tenants.
7. **Profile page**: real name, email, role badge with color, member-since (created_at) shown; effective permissions list populated from `/users/me/permissions`.
8. **Ponytail limitation**: role color null-clear on edit is not supported — setting color to null in the edit dialog sends no update (color field omitted). To clear a role's color, the backend must be updated to support `Option<Option<String>>`.
