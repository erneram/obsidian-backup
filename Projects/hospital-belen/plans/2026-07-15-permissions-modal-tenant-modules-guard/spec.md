# Spec — Fix modal Permisos + provisioning y jerarquía de módulos por tenant

**Proyecto:** hospital-belen · **Stage:** 1 (planner) · **Effort:** medium
**Fecha:** 2026-07-15 · **Estado:** decisiones cerradas, listo para coder.

**DISPATCH:** (1) modal Permisos crece hacia arriba → que crezca hacia abajo.
(2) Provisioning de módulos por tenant. Decisiones del humano:
- Plantilla = **tenant `default`** (Opción A), con **checkbox en create_tenant** para
  usar plantilla.
- Jerarquía **global → admin tenant → users**. **Admin SIN módulo = no lo ve ni lo
  puede asignar.**

---

## Parte 1 — Modal de Permisos crece hacia arriba

### Ubicación
`hospital-belen-web/kairosaid/src/modules/admin/pages/RoleListPage.vue:168-235`
(diálogo "Permisos — <rol>" con `Tabs` por módulo).

### Causa raíz
1. `DialogContent` de `@inksightdev/ui` está `fixed top-[50%] translate-y-[-50%]`
   (centrado) → al crecer el alto se expande desde el centro; el borde superior sube.
2. `TabsList` (línea 180) sin `flex-nowrap` → los triggers hacen *wrap* a varias filas
   cuando hay muchos módulos, sumando alto.

### Cambios (solo `RoleListPage.vue`)

**a) Anclar el diálogo arriba** — línea 170:
```html
<DialogContent class="max-w-2xl top-[6vh] translate-y-0 max-h-[88vh] overflow-hidden flex flex-col">
```
`// ponytail:` verificar que el `cn()` de `@inksightdev/ui` use `tailwind-merge` (la
clase pasada debe ganar sobre `top-[50%]`). Si el merge no gana, fallback
`style="top:6vh;transform:none"`.

**b) `TabsList` en una fila con scroll horizontal** — líneas 180-186:
```html
<TabsList class="mb-3 w-full overflow-x-auto justify-start flex-nowrap">
  <TabsTrigger ... class="whitespace-nowrap shrink-0">
```

### Edge cases
- 1 módulo: sin cambio visible. 10+ módulos: tab-bar scrollea horizontal, contenido
  scrollea dentro de `max-h-72`, modal fijo arriba, footer siempre visible (`max-h-88vh`).

---

## Parte 2 — Provisioning + jerarquía de módulos por tenant

### Contexto / hueco actual
- `menu_items` es **GLOBAL** (sin `tenant_id`, `name` UNIQUE). Solo `menu_item_roles`
  (mapping módulo↔rol) es **per-tenant**, junto con `roles` y `role_permissions`.
- `get_modules` (module.rs) ya filtra la navegación por `menu_item_roles` del rol del
  usuario; PlatformAdmin ve todo. → "no ve" ya funciona.
- **Hueco:** `create_tenant` (tenant.rs:77-88) solo inserta la fila del tenant. No
  provisiona roles/permisos/menús → **clínica nueva nace vacía**. Solo `default` está
  provisionado (seed 002).

### Jerarquía de módulos (decisión cerrada)

```
GLOBAL (platform_super_admin)
  └─ define el catálogo de menu_items (global) y puede asignar cualquiera
        │  provisioning al crear tenant (clona de `default`)
        ▼
ADMIN DE TENANT (tenant_admin)
  └─ posee un subconjunto de módulos (los de menu_item_roles de sus roles)
  └─ SOLO puede ver y asignar módulos que él mismo posee
        │  asigna módulos a roles de su tenant
        ▼
USERS
  └─ reciben módulos vía los roles que el admin les asigna
```

**Regla dura:** un `tenant_admin` sin el módulo X **no lo ve** (ya, vía get_modules) y
**no puede asignárselo a nadie** (nuevo guard backend + filtrado UI).

---

### 2A. Provisioning al crear tenant (clona de `default`)

**Backend**

1. `domain/tenant/mod.rs` — `TenantForCreate` (línea 56): añadir
   ```rust
   #[serde(default = "default_true")] pub use_template: bool,
   ```
   `fn default_true() -> bool { true }`. (camelCase → `useTemplate` desde el front.)

2. `web/handlers/tenant.rs::create_tenant`: tras crear el tenant, si `use_template`,
   ejecutar la rutina de clonado **en una transacción**; si falla, rollback del tenant
   entero (crear tenant + clonar en la misma tx, no dos pasos sueltos).

3. **Rutina de clonado** — `TenantProvisioningService` nuevo en
   `src/domain/tenant/` (o método en `tenant_repository.rs` con `sqlx::Transaction`).
   Fuente = tenant con `slug = 'default'`. Clona **solo** (menu_items NO, son globales):
   - **`roles`**: todas las filas del tenant `default` → nuevas filas con
     `tenant_id = nuevo`, mismos `slug/name/description/is_superadmin/is_system`.
     Guardar mapa `slug → new_role_id`.
   - **`role_permissions`**: para cada `(role, permission)` del `default`, insertar
     `(new_role_id_por_slug, permission_id)`. `permissions` son globales → mismo
     `permission_id`.
   - **`menu_item_roles`**: para cada `(menu_item, role)` del `default`, insertar
     `(menu_item_id, new_role_id_por_slug, tenant_id=nuevo)`. `menu_item_id` global.
   - Todo idempotente con `ON CONFLICT DO NOTHING`.

4. **Migración `migrations/0NN_backfill_tenant_provisioning.sql`**: provisionar tenants
   existentes huérfanos (todos salvo `default`) con las mismas 3 copias, vía SQL puro
   (INSERT … SELECT con join por slug). Idempotente.

**Frontend** — `src/modules/admin/pages/TenantListPage.vue`

- Añadir al form de crear (tras `legalName`, ~línea 75) un `Checkbox`:
  ```html
  <label class="flex items-center gap-2 mt-3 text-sm">
    <Checkbox v-model="form.useTemplate" /> {{ $t('admin.tenants.useTemplate') }}
  </label>
  ```
  `form.useTemplate` default `true`. Hint corto: *"Copia roles y módulos de la clínica
  base. Desmárcalo solo para configurar desde cero."*
- `adminService.createTenant` + `TenantCreate` (adminService.ts): añadir
  `useTemplate?: boolean`.
- i18n `es.json`: `admin.tenants.useTemplate` + hint.

`// ponytail:` una sola plantilla (`default`) — no selector de múltiples plantillas;
el checkbox solo decide *usar plantilla sí/no* (Opción A del humano).

---

### 2B. Asignación de módulos por el admin de tenant (jerarquía)

Nueva capacidad: `tenant_admin` gestiona qué módulos ve cada **rol** de su tenant,
acotado a sus propios módulos.

**Backend**

1. `MenuItemRoleRepository` (`domain/rbac/menu_item_role.rs`): añadir
   ```rust
   async fn set_items_for_role(&self, ctx: &Ctx, role_id: Uuid,
                               menu_item_ids: &[Uuid], tenant_id: Uuid) -> Result<()>;
   ```
   Impl: diff contra el estado actual (`get_items_for_roles(&[role_id], tenant)`) y
   aplicar assign/revoke; o borrar-y-reinsertar dentro de una tx. Scope por `tenant_id`.

2. Endpoints (handler nuevo o en `rbac.rs`), en un **grupo gateado a
   `mw_require_min_role(AppRole::TenantAdmin)`** — NO en el grupo platform-only de
   `/api/roles` CRUD:
   - `GET  /api/roles/{id}/menu-items` → `{ data: [menu_item_id,...] }`.
   - `PUT  /api/roles/{id}/menu-items` body `{ menu_item_ids: [...] }`.

3. **Guard de jerarquía** en el handler `PUT` (y filtrado en `GET`):
   - Verificar que el `role_id` pertenece a `ctx.tenant_id()` (si no → 404/403).
   - Si `!ctx.is_platform_admin()`:
     - Calcular el set propio del caller:
       `own = get_items_for_roles(get_user_roles(ctx.user_id, ctx.tenant_id).ids, ctx.tenant_id)`.
     - Rechazar (`403 Forbidden`) si algún `menu_item_id` del body **no** está en `own`.
       → "admin sin módulo no puede asignarlo".
   - PlatformAdmin: sin restricción de subconjunto.

**Frontend** — `src/modules/admin/pages/RoleListPage.vue`

- Añadir acción **"Módulos"** por fila de rol (junto a "Permisos", ~línea 71-73),
  abriendo un modal nuevo.
- El modal lista los **módulos asignables del caller**, obtenidos de **`GET
  /api/modules`** (ya devuelve el árbol filtrado a lo que el caller ve → es exactamente
  su set asignable; PlatformAdmin ve todo). Estado actual del rol desde
  `GET /api/roles/{id}/menu-items`; guardar con el `PUT`.
- Checkboxes por módulo (reusar el patrón de checkboxes del modal de Permisos; agrupar
  por sección padre del árbol de `/api/modules`). Aplicar el **mismo anclaje/scroll de
  Parte 1** a este modal desde el inicio.
- `adminService`: `getRoleMenuItems(id)` y `setRoleMenuItems(id, ids)`.

---

## Archivos afectados (para el coder)

| Parte | Archivo | Cambio |
|---|---|---|
| 1 | `web/.../admin/pages/RoleListPage.vue` | DialogContent top-anchor + TabsList flex-nowrap |
| 2A | `api/src/domain/tenant/mod.rs` | `use_template` en TenantForCreate |
| 2A | `api/src/web/handlers/tenant.rs` | clonado tx en create_tenant |
| 2A | `api/src/domain/tenant/` (service nuevo) o `.../repositories/tenant_repository.rs` | rutina de clonado |
| 2A | `api/migrations/0NN_backfill_tenant_provisioning.sql` | backfill tenants huérfanos |
| 2A | `web/.../admin/pages/TenantListPage.vue` + `services/adminService.ts` + `locales/es.json` | checkbox plantilla |
| 2B | `api/src/domain/rbac/menu_item_role.rs` (+ repo impl) | `set_items_for_role` |
| 2B | `api/src/web/handlers/rbac.rs` + `router.rs` | endpoints menu-items por rol, grupo TenantAdmin + guard subconjunto |
| 2B | `web/.../admin/pages/RoleListPage.vue` + `services/adminService.ts` | modal "Módulos" |

---

## Edge cases (Parte 2)

- **create_tenant sin plantilla** (`useTemplate=false`): tenant vacío, sin roles/menús —
  comportamiento intencional; el platform admin configura a mano.
- **Clonado parcial fallido**: la tx garantiza atomicidad (todo o nada). Sin estados
  a medias.
- **`default` no existe / renombrado**: el clonado debe abortar con error claro
  ("plantilla `default` no encontrada"), no crear tenant vacío en silencio.
- **Guard 2B**: admin envía un `menu_item_id` fuera de su set → `403`, no ignorar
  silenciosamente (evita escalada de módulos).
- **Rol de otro tenant** en `PUT /roles/{id}/menu-items` → 404/403 (aislar tenants).
- **PlatformAdmin** usando 2B: sin restricción de subconjunto (nivel global).
- **Backfill idempotente**: correr la migración dos veces no duplica
  (`ON CONFLICT DO NOTHING`).
- **Secciones padre**: al asignar un módulo hijo, asegurar que el padre (section-*)
  también quede visible o `get_modules` no renderiza el grupo. Incluir los `parent_id`
  de los items seleccionados en el set guardado. `// ponytail:` derivar padres del
  árbol, no pedirlos al usuario.

---

## Patrones a seguir
- Gate de rol por grupo: bloque `tenants` en `router.rs` (`mw_require_min_role`).
- Filtrado módulo↔rol: `module.rs::get_modules` + `get_items_for_roles`.
- Fuente del clonado: seed `002_seed_rbac.sql` (estructura de roles/role_permissions/
  menu_item_roles del tenant `default`).
- Transacción atómica: repos que ya usan `sqlx::Transaction` en `repositories/`.

## SKILL_RECOMENDADA
Ninguna. CSS puntual (P1) + provisioning/guards sobre el modelo RBAC propio (P2).

## Fuera de alcance
- Selector de múltiples plantillas (solo `default`, decisión del humano).
- Asignación de módulos **por usuario individual** (el modelo es módulo↔rol; el usuario
  hereda vía roles). Si se quiere granularidad por usuario, es feature aparte.
- Tests (stage-3). Casos a cubrir: modal 10+ módulos no desborda; create_tenant con
  plantilla deja roles+menús; tenant nuevo → usuario ve sus módulos; admin sin módulo X
  recibe 403 al intentar asignar X.
