# Changes — hospital-belen · stage-2 (tenant_admin module scope)

## Archivos modificados / creados

### `hospital-belen-api/src/web/router.rs`
- Divide el grupo `rbac` en dos:
  - **`rbac_platform`** (`PlatformAdmin+`): permissions, roles CRUD, endpoints, endpoint-groups, menu-items, audit-logs. Gate: `mw_require_min_role(AppRole::PlatformAdmin)`.
  - **`rbac_tenant`** (`TenantAdmin+`): `GET/POST /api/users/{id}/roles`, `DELETE /api/users/{id}/roles/{role_id}`. Gate: `mw_require_min_role(AppRole::TenantAdmin)`.
- Assembler actualizado: `.merge(rbac_platform).merge(rbac_tenant)` reemplaza `.merge(rbac)`.

### `hospital-belen-api/src/web/handlers/module.rs`
- `get_modules` recibe `ctx: Ctx` (antes sin contexto).
- PlatformAdmin → sin filtro (ve todo).
- Otros → consulta `user_role_repo.get_user_roles` + `menu_item_role_repo.get_items_for_roles` → filtra `items` al `HashSet<Uuid>` permitido.
- Grupos padre sin hijos visibles se eliminan del response (`retain`).

### `hospital-belen-api/migrations/031_tenant_admin_module_scope.sql` (NUEVO)
- Elimina `menu_item_roles` de `tenant_admin` para `roles-list`, `endpoints`, `admin-tenants`.
- Idempotente (DELETE por join en name/slug).
- Comentario ponytail: desacopla visibilidad de navegación del flag `is_superadmin`.

## Sin cambios
- Frontend: sidebar ya consume `/api/modules` dinámicamente; no hay links hardcodeados a `/admin/roles|endpoints|tenants`.
- Route-guards `requiresPlatformAdmin` ya existían en `admin/router.ts`.
- Backend operativo: `is_superadmin` de tenant_admin sigue bypasseando permisos en rutas de pacientes, citas, etc.

## Qué revisar puntualmente (Tester)

1. **403 platform-only**: tenant_admin haciendo `GET /api/roles` → 403. `GET /api/endpoints` → 403. `GET /api/audit-logs` → 403.
2. **200 tenant-admin**: tenant_admin haciendo `GET /api/users/{id}/roles` → 200.
3. **Sidebar filtrado**: usuario con rol `tenant_admin` no recibe `roles-list`, `endpoints`, `admin-tenants` en `GET /api/modules`.
4. **Sidebar platform_admin**: PlatformAdmin sigue viendo todos los módulos.
5. **Sidebar usuario operativo** (doctor, recepcionista): solo ve módulos seeded para su rol.
6. **Grupos vacíos**: si un grupo padre pierde todos sus hijos, no aparece en el response.
7. **Regresión**: rutas operativas (pacientes, citas, talonarios) siguen funcionando para tenant_admin.
