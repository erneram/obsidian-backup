# Spec — Rediseño modal /platform/endpoints + Resource Relationships

Decisión OPEN QUESTION: **opción B**. "Relevante" = resources definidos en una tabla
curada `resource_relationships`. Se añade una vista de admin para mantenerla.

## Alcance (3 partes)
1. **Backend**: tabla `resource_relationships` + CRUD endpoints platform.
2. **Frontend modal endpoints**: quitar headers de tabla + filtrar permisos por relaciones.
3. **Frontend vista nueva**: `/platform/resource-relationships` para mantener el mapa.

Stack: API en Rust (axum + sea-query/modql BMC), Web en Vue 3 + `@inksightdev/ui`.

---

# PARTE 1 — Backend (hospital-belen-api)

## Migración
Crear `migrations/004_resource_relationships.sql` (nueva, no editar 001-003):
```sql
CREATE TABLE resource_relationships (
  id                uuid         PRIMARY KEY DEFAULT gen_random_uuid(),
  resource          varchar(100) NOT NULL,   -- ej: admissions
  related_resource  varchar(100) NOT NULL,   -- ej: billing
  created_at        timestamptz  NOT NULL DEFAULT now(),
  UNIQUE (resource, related_resource)
);
CREATE INDEX idx_resource_relationships_resource ON resource_relationships (resource);
```
Modelo: filas `(resource, related_resource)`. Un resource "admissions" con relacionados
`[admissions, appointments, billing]` = 3 filas (incluye sí mismo, o el backend lo agrega
implícito — ver regla abajo).

**Regla self-inclusion**: el resource propio SIEMPRE es relevante. Decidir uno:
- (A) Guardar la fila `(admissions, admissions)` explícita, o
- (B) No guardarla y que el frontend/handler agregue el self siempre.
Preferir **(B)**: menos datos redundantes, self implícito. La tabla guarda solo relaciones extra.

## Domain (mirar patrón `src/domain/rbac/permission.rs`)
Crear `src/domain/rbac/resource_relationship.rs`:
- Struct `ResourceRelationship { id, resource, related_resource, created_at }` (Serialize, FromRow).
- Trait `ResourceRelationshipRepository` con:
  - `list_all(&self) -> Result<Vec<ResourceRelationship>>`
  - `set_for_resource(&self, resource: String, related: Vec<String>) -> Result<()>`
    (borra las filas del resource y reinserta — mismo patrón que `endpoint_repo.set_permissions`).
- Registrar `pub mod resource_relationship;` en `src/domain/rbac/mod.rs`.

## Repo (mirar `permission_repository.rs` — BMC + Postgres adapter)
Crear `src/infrastructure/database/repositories/resource_relationship_repository.rs`:
- `ResourceRelationshipBmc { TABLE="resource_relationships", HAS_TIMESTAMPS=false, DELETE_MODE=Hard, IS_TENANT_SCOPED=false }`.
- `PostgresResourceRelationshipRepository { db: Db }` implementa el trait.
- `set_for_resource`: en una tx, `DELETE FROM resource_relationships WHERE resource=$1`
  luego insert de cada related. Reusar helpers `base`/BMC como hace endpoint_repo.
- Registrar en `repositories/mod.rs`: `pub mod ...` + `pub use ...Postgres...`.

## AppState (src/infrastructure/mod.rs)
- Añadir campo `pub resource_relationship_repo: Arc<dyn ResourceRelationshipRepository>` (~línea 86).
- Instanciar (~línea 126) y pasar al struct final (~línea 202). Mismo patrón que `permission_repo`.

## Handlers (src/web/handlers/platform_rbac.rs — mirar list_permissions_platform:430)
Añadir:
```
GET /api/platform/resource-relationships
  → list_resource_relationships_platform
  → devuelve { "data": { "<resource>": ["<related>", ...], ... } }  (map agrupado)
    Agrupar las filas por resource en el handler.

PUT /api/platform/resource-relationships/{resource}
  → set_resource_relationships_platform
  → body: { "related": ["appointments", "billing"] }
  → 204 No Content
```
- Reusar `Ctx`, `State<AppState>`, `WebError::Domain`. Filtrar el self del `related` entrante
  (no guardar `(admissions, admissions)`) — coherente con regla (B).

## Router (src/web/router.rs — bloque rbac_platform, junto a línea 732)
```rust
.route(
    "/api/platform/resource-relationships",
    get(platform_rbac_handlers::list_resource_relationships_platform),
)
.route(
    "/api/platform/resource-relationships/{resource}",
    put(platform_rbac_handlers::set_resource_relationships_platform),
)
```
Dentro del mismo grupo con `mw_platform_ctx_require` (auth platform).

## Edge cases backend
- Resource sin relaciones → no aparece en el map (frontend hace fallback a `[self]`).
- `related` vacío en PUT → borra todas las relaciones extra del resource (deja solo self implícito). OK.
- `related` incluye el propio resource → filtrarlo antes de insertar.
- Resource inexistente en `permissions` → **no validar** contra permissions; la tabla es libre
  (los resources salen de los paths, no hay FK). `// ponytail: sin FK, resources son strings de paths`.

---

# PARTE 2 — Frontend modal endpoints (hospital-belen-web)

## `.../platform/services/platformAdmin.service.ts`
Añadir 2 métodos (mirar `getEndpointPermissions`:206 / `setEndpointPermissions`:215):
```ts
async listResourceRelationships() {
  // GET /platform/resource-relationships → { data: Record<string, string[]> }
}
async setResourceRelationships(resource: string, related: string[]) {
  // PUT /platform/resource-relationships/{resource}  body { related }
}
```

## `.../platform/components/ModuleGroupCard.vue`
- **Quitar `<TableHeader>`** (líneas 10-18) — headers "Método/Ruta/Descripción/Permiso".
- Mantener card header con color de módulo (líneas 3-7). Filas sin cambios.
- Verificar alineación sin header; si queda raro, degradar a lista flex (ver Diseño visual).

## `.../platform/pages/PlatformEndpointsPage.vue`
- Cargar el map de relaciones en `onMounted` (`Promise.all` línea 143):
  ```ts
  const relationships = ref<Record<string, string[]>>({})
  async function loadRelationships() {
    const res = await platformAdminService.listResourceRelationships()
    if (res.success) relationships.value = (res.data as { data?: Record<string,string[]> }).data ?? {}
  }
  ```
- Helper `relatedResources(resource)`:
  ```ts
  function relatedResources(r: string): string[] {
    return [r, ...(relationships.value[r] ?? [])]  // self siempre incluido
  }
  ```
- `relevantPermissions` computed (reemplaza `allPermissions` en el modal):
  ```ts
  const showAllPerms = ref(false)
  const relevantPermissions = computed<Permission[]>(() => {
    if (showAllPerms.value || !epPermsTarget.value) return allPermissions.value
    const rels = new Set(relatedResources(moduleOf(epPermsTarget.value.path)))
    return allPermissions.value.filter(p => rels.has(p.resource) || epSelectedPerms.value.has(p.id))
  })
  ```
  - `|| epSelectedPerms.has(p.id)`: **crítico** — permiso ya asignado de resource no relacionado
    debe seguir visible para poder desmarcarlo.
- Reset `showAllPerms=false` al abrir modal (`openEndpointPerms`:242).
- Template modal (línea 96): `v-for="p in relevantPermissions"`, agrupado por `resource`.
- Toggle "Ver todos ({{ allPermissions.length }})" ↔ "Ver relevantes" en el body.
- `saveEpPerms`:263 **no cambia** (usa el set completo, no el filtrado).

---

# PARTE 3 — Frontend vista Resource Relationships (nueva)

## Archivo nuevo `.../platform/pages/PlatformResourceRelationshipsPage.vue`
Vista simple para mantener el map. **No compleja** (requisito explícito).

Fuentes de resources para el selector:
- Derivar la lista de resources de `listPermissions()` (`resource` distinct) — es la
  fuente natural. No hace falta endpoint nuevo para listar resources.

Comportamiento:
- Lista todos los resources (distinct de permissions).
- Por cada resource: muestra chips de sus related actuales + input/select para añadir/quitar.
- Guardar por resource → `setResourceRelationships(resource, related)`.
- Guardado optimista o por fila; toast en éxito (patrón `toast.success` ya usado).

Estado: `relationships: Record<string,string[]>`, `resources: string[]`.
Cargar ambos en `onMounted`.

## Router `.../platform/router.ts` (junto a endpoints, línea 102)
```ts
{
  path: 'resource-relationships',
  name: 'platform-resource-relationships',
  component: () => import('./pages/PlatformResourceRelationshipsPage.vue'),
  meta: { title: 'Relaciones de Recursos' },
},
```

## Nav `.../layouts/PlatformLayout.vue` (junto al link de endpoints, línea 101-112)
Añadir un `<RouterLink to="/platform/resource-relationships">` con el mismo markup,
un icono lucide (ej `Share2` o `GitFork`), label `{{ $t('platform.nav.resourceRelationships') }}`.
Añadir la key i18n `platform.nav.resourceRelationships` (buscar el archivo de traducción
que define `platform.nav.endpoints` y agregar la key en es/en).

---

## Diseño visual

**Modal endpoints** (foco por resource):
```
┌─ Permisos — GET /api/admissions ──────────────┐
│                                    [Ver todos] │
│  admissions                                    │
│    ☑ admissions:read    Ver admisiones         │
│    ☐ admissions:create  Crear admisión         │
│  appointments        (relacionado)             │
│    ☐ appointments:read  Ver citas              │
│  billing             (relacionado)             │
│    ☐ billing:read       Ver facturación        │
│              [Cancelar]  [Guardar]             │
└────────────────────────────────────────────────┘
```
- Subheader por resource: `capitalize font-medium text-xs text-muted-foreground`.
- Badge de permiso: reusar `font-mono text-xs bg-blue-100 text-blue-800` (línea 105-107).

**Vista Resource Relationships** (simple, chips editables):
```
┌─ Relaciones de Recursos ───────────────────────┐
│  Define qué recursos son relevantes al asignar  │
│  permisos por endpoint.                         │
│                                                 │
│  admissions   [appointments ×] [billing ×] [+]  │
│  appointments [admissions ×] [+]                │
│  billing      [+]                               │
│  inventory    [+]                               │
└─────────────────────────────────────────────────┘
```
- Cada fila: nombre del resource (mono) + chips de related con botón ×  + botón/select "+"
  para añadir (opciones = otros resources).
- Chip: `inline-flex items-center gap-1 px-2 py-0.5 rounded bg-muted text-xs font-mono`.
- Guardar por fila al cambiar; toast breve. Sin tabla, sin modal — lista directa.

---

## SKILL_RECOMENDADA
Ninguna.

## Notas para coder / tester
- Effort medium. Orden sugerido: migración → domain/repo/AppState → handlers/router →
  build API (`cargo check`) → service web → modal → vista nueva → router+nav+i18n.
- Backend: seguir EXACTO el patrón de `permission`/`endpoint` (BMC + trait + Arc en AppState).
  No inventar una capa nueva.
- Verificar: (a) modal muestra solo related+self+asignados, toggle revela todos;
  (b) permiso asignado de resource no-related sigue visible; (c) vista relationships
  persiste y se refleja en el filtro del modal.
- Seed opcional para probar: insertar `(admissions, billing)` y validar el filtro.
