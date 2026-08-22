# Spec — Control de acceso a módulos administrativos por rol

**Proyecto:** hospital-belen · **Stage:** 1 (planner) · **Effort:** medium
**Fecha:** 2026-07-10
**DISPATCH:** Admin de clínica no debe ver Roles/Endpoints/Tenants. Validar en backend
los módulos disponibles según el rol del usuario.

---

## OPEN QUESTION (resolver antes de codear)

1. **¿El admin de clínica (`tenant_admin`) asigna roles a los usuarios de su clínica?**
   El endpoint `POST/DELETE /api/users/{id}/roles` (asignar/revocar rol a usuario) hoy
   está en el mismo grupo sin gate. Dos opciones:
   - **(A, asumida)** tenant_admin SÍ asigna roles existentes a sus usuarios → separar
     ese sub-grupo y gatearlo a `TenantAdmin`; el catálogo de roles (`/api/roles` CRUD)
     queda platform-only.
   - **(B)** todo lo RBAC es platform-only → tenant_admin no toca roles en absoluto.
2. **¿`audit-logs` visible para tenant_admin?** Asumido: **sí** (auditoría de su clínica).
   Se deja fuera del gate platform-only.

---

## Hallazgo crítico (hay un hueco de seguridad real, no solo cosmético)

El grupo de rutas RBAC en `hospital-belen-api/src/web/router.rs` **líneas 344–427**
(`/api/roles`, `/api/endpoints`, `/api/endpoint-groups`, `/api/menu-items`,
`/api/audit-logs`, `/api/users/{id}/roles`) está protegido **solo por
`mw_ctx_require`** (línea 427) — es decir, **cualquier usuario autenticado**
(recepcionista incluido) puede listar/crear/borrar roles, endpoints y menús.

Comparar:
- `tenants` (línea 545) → gate `mw_require_min_role(AppRole::PlatformAdmin)`. ✅
- `users` (línea 94+) → gate `mw_require_permission` por ruta. ✅
- `rbac` (línea 344) → **sin gate de rol/permiso.** ❌

**Esta es la validación de backend que pide el DISPATCH.** No existe hoy.

Además, en el frontend los route-guards de `/admin/roles`, `/admin/endpoints`,
`/admin/tenants` ya son `requiresPlatformAdmin` (`src/modules/admin/router.ts`) — la
capa de navegación cliente ya está bien; el hueco es **servidor + sidebar**.

---

## Segundo hallazgo — el sidebar filtra nada

`GET /api/modules` → `module.rs::get_modules` llama `menu_item_repo.get_all_active()`:
devuelve **todos** los menu_items activos, **sin `Ctx`** y **sin mirar
`menu_item_roles`**. El sidebar (`module.store` → `/api/modules`) muestra Roles,
Endpoints, Tenants a todo el mundo aunque el route-guard luego bloquee la navegación.

La infraestructura para filtrar **ya existe** y no se usa:
- Tabla `menu_item_roles` + seed por rol (`002_seed_rbac.sql`).
- `MenuItemRoleRepository::get_items_for_roles(role_ids, tenant_id) -> Vec<Uuid>`
  (`src/domain/rbac/menu_item_role.rs:14`).
- `UserRoleRepository::get_user_roles(user_id, tenant_id) -> Vec<Role>`
  (`src/domain/rbac/user_role.rs:10`) — trae los role UUIDs + `is_superadmin`.

---

## Tercer hallazgo — el seed da TODO a tenant_admin

`002_seed_rbac.sql` línea ~537: para el rol slug `admin` (renombrado a `tenant_admin`
por `026_rename_roles.sql`) hace `CROSS JOIN menu_items` → le asigna **todos** los
menu_items, incluidos `roles-list`, `endpoints`, `admin-tenants`.

Y peor: el rol `admin` se siembra con **`is_superadmin = true`** (línea 97-98). Si el
filtro de `get_modules` hiciera bypass para superadmins, tenant_admin seguiría viéndolo
todo. → **El filtro de módulos debe basarse en `AppRole::PlatformAdmin`, NO en
`is_superadmin`.** Se desacopla "bypass de permisos de API" (is_superadmin) de
"visibilidad de módulos" (nivel de rol).

---

## Cambios (rutas relativas al repo)

### Backend — `hospital-belen-api/`

#### 1. `src/web/router.rs` — gatear RBAC platform-only (PRIMARIO)
Partir el grupo actual (344–427) en dos:

- **Grupo platform-only** → `roles`, `endpoints`, `endpoint-groups`, `menu-items`.
  Añadir `route_layer(from_fn_with_state(AppRole::PlatformAdmin, mw_require_min_role))`
  igual que el grupo `tenants` (556–559).
- **Grupo tenant-admin** (según OPEN QUESTION 1/2) → `users/{id}/roles` (asignar/revocar)
  + `audit-logs`. Gate `mw_require_min_role(AppRole::TenantAdmin)`.

Patrón exacto a copiar: bloque `tenants` líneas 545–560.

#### 2. `src/web/handlers/module.rs` — filtrar módulos por rol
`get_modules` pasa a recibir `ctx: Ctx`:
```rust
pub async fn get_modules(ctx: Ctx, State(state): State<AppState>)
    -> Result<Json<Vec<ModuleGroupResponse>>>
```
Lógica:
1. `let items = state.menu_item_repo.get_all_active().await?;`
2. Si `ctx.role().is_platform_admin()` → no filtrar (ve todo). *(usar
   `ctx.is_platform_admin()`, ya existe en `auth/ctx.rs:44`)*.
3. Si no:
   - `roles = state.user_role_repo.get_user_roles(ctx.user_id(), ctx.tenant_id())`
   - `allowed: HashSet<Uuid> = state.menu_item_role_repo
        .get_items_for_roles(&role_ids, ctx.tenant_id()).await?` en set.
   - Filtrar `items` a los que estén en `allowed`.
4. **Incluir secciones padre** cuya al menos un hijo quede permitido (para que el grupo
   renderice) y **descartar grupos vacíos** tras armar `groups`.

`AppState` ya expone `user_role_repo` y `menu_item_role_repo` (verificar nombres en
`src/infrastructure/mod.rs`; si falta alguno, añadirlo al DI).

#### 3. Migración nueva — `migrations/0NN_tenant_admin_module_scope.sql`
No editar `002` (idempotente, ya seedeado). Crear migración que:
- `DELETE FROM menu_item_roles` para el rol `tenant_admin` en los items
  `roles-list`, `endpoints`, `admin-tenants` (y `menu-items` si existiera en el seed).
- Idempotente (borrado por join a `menu_items.name` + `roles.slug='tenant_admin'`).
- Comentario `-- ponytail:` explicando que tenant_admin conserva is_superadmin para
  API pero pierde estos módulos de navegación (visibilidad ≠ bypass).

`OPEN QUESTION:` si se decide (B), incluir también quitar cualquier menú RBAC extra.

### Frontend — `hospital-belen-web/kairosaid/` (mínimo)

Ninguna lógica nueva necesaria: el sidebar ya se arma desde `/api/modules`
(`module.store.ts`) — al filtrar el backend, los ítems desaparecen solos. Los
route-guards `requiresPlatformAdmin` ya bloquean navegación directa.

- **Verificar** que no haya links hardcodeados a `/admin/roles|endpoints|tenants` fuera
  del sidebar dinámico (grep en `src/layouts/`, `src/modules/admin/`). Si los hay,
  envolver en `v-if="authStore.isPlatformAdmin"`.

---

## Interfaces / firmas

```rust
// module.rs
pub async fn get_modules(ctx: Ctx, State(state): State<AppState>) -> Result<Json<Vec<ModuleGroupResponse>>>
// usa:
ctx.is_platform_admin() -> bool                              // ya existe
UserRoleRepository::get_user_roles(user_id, tenant_id)       // ya existe
MenuItemRoleRepository::get_items_for_roles(&[Uuid], Uuid)   // ya existe
```
Sin cambios de tipos de respuesta (`ModuleGroupResponse` intacto). Sin cambios frontend.

---

## Edge cases

- **platform_super_admin** (is_platform_admin): ve todos los módulos, todas las rutas RBAC. ✅
- **tenant_admin**: ve operativos + Usuarios + Auditoría; NO Roles/Endpoints/Tenants.
- **Usuario con múltiples roles**: unión de menu_items permitidos (`get_items_for_roles`
  ya acepta lista).
- **Rol sin `menu_item_roles`**: sidebar vacío salvo dashboard — confirmar que
  `dashboard` está sembrado para todos los roles operativos (el seed lo incluye).
- **Grupo padre sin hijos permitidos**: no renderizar (evitar secciones vacías).
- **Regresión de API**: tras gatear, tenant_admin que hoy llamaba `/api/roles` recibirá
  403 — es el comportamiento deseado; el tester debe cubrirlo con un caso 403.
- **is_superadmin de tenant_admin**: sigue dando bypass de permisos en endpoints
  operativos; el gate de módulos usa nivel de rol, no el flag → no se rompe.

---

## Patrones existentes a seguir

- Gate de rol: bloque `tenants` en `router.rs:545-560` (`mw_require_min_role`).
- Handler con `Ctx` + repos vía `State(state)`: cualquier handler en `rbac.rs`.
- Filtrado por rol vía menu_item_roles: ya diseñado, solo hay que **usarlo**.

---

## SKILL_RECOMENDADA
Ninguna. Es cableado de middleware Axum + una query de filtrado sobre el modelo RBAC
propio. No hay skill instalable aplicable.

---

## Fuera de alcance
- No convertir `tenant_admin` en no-superadmin (blast radius enorme: requeriría sembrar
  todos los `role_permissions` operativos). Se mantiene is_superadmin; se controla solo
  la **visibilidad de módulos** y el **gate de rutas platform**.
- No rediseño de UI (esto es control de acceso, no la feature de creación de roles del
  spec anterior).
- No tests (los escribe el tester en stage-3); este spec solo nombra los casos 403 a cubrir.
