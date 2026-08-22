# Spec — Creación de roles por módulo en Administración

**Proyecto:** hospital-belen · **Stage:** 1 (planner) · **Effort:** medium
**Fecha:** 2026-07-10

---

## OPEN QUESTION (resolver antes de codear)

"usuario base = admin en cada módulo" admite dos lecturas. Elegí una:

- **(A) Plantilla admin-total** *(asumida por defecto en este spec)*: al crear un rol,
  arranca con **todos** los permisos de **todos** los módulos marcados (admin en cada
  uno) y el operador *desmarca* lo que sobra. Grant rápido "Admin de este módulo" por
  pestaña.
- **(B) Admin por módulo real**: cada módulo tiene su propio rol-admin independiente y
  se asigna admin módulo-por-módulo a un usuario (matriz usuario × módulo).

El spec implementa **(A)** porque no requiere cambios de schema y reusa el RBAC actual.
Si el humano quiere (B), avisar — cambia el modelo de datos.

---

## Contexto RBAC actual (revisión)

El backend RBAC **ya está completo**. No requiere cambios para (A).

- **Schema** (`hospital-belen-api/migrations/001_schema.sql`): `permissions(resource,
  action)`, `roles`, `role_permissions`, `user_roles`, `endpoints`,
  `endpoint_permissions`, `menu_items`, `menu_item_roles`. Roles con `is_superadmin`
  (bypass total) y scope por `tenant_id`.
- **Permisos** agrupados de facto por `resource` (`patients`, `billing`, `inventory`,
  `hr`, `pediatrics`, `appointments`, `rbac`, `users`, …) → **cada `resource` = un
  "módulo"** para la UI.
- **API roles** (`src/web/handlers/rbac.rs`): `GET/POST /roles`, `PUT/DELETE
  /roles/:id`, `GET/PUT /roles/:id/permissions`, `GET /permissions`. Todo lo que la UI
  necesita ya existe.
- **Frontend** (`src/modules/admin/pages/RoleListPage.vue`): ya tiene grid + modales de
  crear/editar/eliminar + modal de permisos (lista plana con checkboxes). **Falta:
  agrupación por módulo y pestañas.**

**Conclusión:** el trabajo es **solo frontend** — reorganizar la selección de permisos
por módulo, con pestañas y grants rápidos. Sin migraciones, sin endpoints nuevos.

---

## Archivos a modificar/crear (rutas relativas al repo)

Todo bajo `hospital-belen-web/kairosaid/`.

### 1. `src/modules/admin/pages/RoleListPage.vue` (MODIFICAR)
Reemplazar el modal de permisos de lista plana por vista **tabulada por módulo**.

- Al abrir el modal de permisos (`openPermissions`), agrupar `allPermissions` por
  `resource` → `Record<string, Permission[]>` (computed).
- Envolver el contenido en `Tabs` de `@inksightdev/ui`:
  - `TabsList`: una `TabsTrigger` por módulo (label legible del `resource`), con badge
    del conteo seleccionado `n/total`.
  - `TabsContent` por módulo: grid de checkboxes de acciones (`read/create/update/…`).
- Header de cada pestaña: checkbox **"Admin de este módulo"** (tri-state visual) que
  marca/desmarca todos los permisos de ese `resource` de una vez.
- Reutilizar `selectedPerms: Set<string>`, `togglePerm`, `savePermissions` existentes.

### 2. Modal "Nuevo Rol" — plantilla base admin (MODIFICAR, mismo archivo)
- Añadir checkbox en `createForm`: **"Admin en todos los módulos"** (default: on →
  interpretación A).
- Tras `createRole` exitoso, si está activo: `PUT /roles/:id/permissions` con **todos**
  los `permission_ids`. Requiere cargar `allPermissions` en `onMounted` (hoy solo se
  cargan al abrir el modal de permisos) → moverlo a carga inicial o cargar on-demand
  antes del PUT.

### 3. `src/modules/admin/components/PermissionMatrix.vue` (CREAR, opcional)
Extraer la grilla-por-pestaña a un componente si `RoleListPage.vue` supera ~450 líneas
tras el cambio. Props: `permissions`, `v-model:selected` (Set). Si no supera, dejarlo
inline. `// ponytail:` no crear el componente hasta que el archivo lo pida.

### 4. `src/locales/es.json` (MODIFICAR)
Añadir claves i18n para labels de módulos y textos nuevos. **No hardcodear strings**
(regla del repo). Mapa `resource → label` (`patients`→"Pacientes", `billing`→
"Facturación", etc.) — ver tabla en Diseño visual.

---

## Interfaces / firmas

```ts
// RoleListPage.vue <script setup>
const permsByModule = computed<Record<string, Permission[]>>(() => …) // group by resource
function moduleLabel(resource: string): string   // resource → i18n label
function isModuleFull(resource: string): boolean  // todos los perms del módulo en selectedPerms
function toggleModule(resource: string, on: boolean): void // grant/revoke módulo entero
// createForm gana: adminAllModules: boolean
```

Sin cambios de tipos backend. `Permission` ya definido inline en el page.

---

## Patrones existentes a seguir

- **Tabs**: ya usados en `src/modules/patients/pages/PatientDetailPage.vue` — copiar
  ese patrón de `Tabs/TabsList/TabsTrigger/TabsContent`.
- **Modales/grid/toast**: el propio `RoleListPage.vue` ya es el patrón canónico
  (`Dialog`, `Table`, `toast`, `ConfirmDialog`).
- **Servicio**: extender `src/modules/admin/services/adminService.ts` solo si hace
  falta un helper de permisos; hoy el page llama `apiService` directo — mantener.
- **i18n**: todo string vía `$t()`, archivo `src/locales/es.json`.

---

## Edge cases a cubrir

- Rol `is_superadmin`: no editable/permisos deshabilitados (ya se oculta en el grid).
- Módulo sin permisos seleccionados vs. todos → estado del checkbox "Admin de módulo"
  (vacío / indeterminado / lleno).
- `allPermissions` vacío → mensaje existente "No hay permisos creados".
- Crear rol con "Admin en todos los módulos" pero el PUT de permisos falla → el rol
  quedó creado sin permisos: mostrar toast de warning claro, no error silencioso.
- Slug duplicado / nombre vacío → validación ya existe, conservar.
- Scope tenant: `list_roles` ya filtra por `ctx.tenant_id()`; no exponer roles de otros
  tenants.

---

## Diseño visual (## Diseño visual)

Panel **interno de administración**, no landing. Regla rectora: **cero identidad
nueva** — se reusa el design system existente (`@inksightdev/ui` shadcn-style + Tailwind
tokens `foreground/muted/card/border/destructive`). El valor UX está en *estructura*,
no en estética novedosa. Ponytail aplicado: la disciplina es consistencia, no adorno.

### Modal de permisos — de lista plana a matriz tabulada

```
┌─ Permisos — Recepcionista ─────────────────────────────┐
│ [Pacientes 2/4] [Citas 4/4] [Facturación 0/3] [Inv…]   │  ← TabsList (scroll-x)
│ ────────────────────────────────────────────────────── │
│ ☑ Admin de este módulo (Pacientes)                     │  ← grant rápido, arriba
│ ────────────────────────────────────────────────────── │
│ ☑ patients:read     Ver pacientes                      │  ← grid de acciones
│ ☑ patients:create   Registrar pacientes                │
│ ☐ patients:update   Editar datos                       │
│ ☐ patients:delete   Eliminar                           │
│ ────────────────────────────────────────────────────── │
│                              [Cancelar]  [Guardar]      │
└────────────────────────────────────────────────────────┘
```

- **Pestaña = módulo** (`resource`). Badge `n/total` en cada trigger comunica cobertura
  de un vistazo — la información *es* la estructura, no decoración.
- **Grant "Admin de este módulo"** fijo arriba de cada pestaña: la acción más común
  (dar admin de un módulo) a un clic. Checkbox indeterminado cuando hay selección
  parcial (feedback honesto del estado).
- Acción `read` primero, luego `create/update/delete/otros` — orden de menor a mayor
  privilegio, lectura como base.

### Mapa `resource → label` (i18n, es.json)

| resource | label |
|---|---|
| `users` | Usuarios |
| `rbac` | Roles y permisos |
| `patients` | Pacientes |
| `appointments` | Citas |
| `medical_records` | Historiales |
| `prescriptions` | Recetas |
| `admissions` | Ingresos |
| `billing` | Facturación |
| `inventory` | Inventario |
| `hr` | Recursos Humanos |
| `reports` | Reportes |
| `pediatrics` | Pediatría |
| `doctors` | Médicos |

Resource sin entrada → fallback a `capitalize(resource)`.

### Modal "Nuevo Rol"

Sin rediseño: agregar una fila con checkbox **"Admin en todos los módulos"** (default
activado) + hint corto: *"El rol arranca con acceso total; ajusta después en Permisos."*
Copy en voz activa, describe qué pasa.

### Tono / copy

- Botones = acción concreta: "Guardar", "Crear", no "Enviar".
- Estado vacío ya resuelto ("No hay permisos creados").
- Sin emojis, sin mayúsculas gritadas; sentence case, consistente con el resto del panel.

---

## SKILL_RECOMENDADA

Ninguna. `npx skills find` no aplica: la matriz de permisos está acoplada al schema RBAC
propio (resource:action). No hay skill instalable que lo cubra; es UI bespoke sobre
componentes ya presentes (`Tabs` de `@inksightdev/ui`).

---

## Fuera de alcance (no hacer)

- Cambios de backend / migraciones (RBAC ya soporta todo para interpretación A).
- Interpretación B (admin-por-módulo real) hasta que el humano lo confirme.
- Reorganizar el resto de páginas de Administración en pestañas — el DISPATCH pide
  creación de roles, no rediseño global del área admin.
