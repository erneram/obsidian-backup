# Spec — Paginación backend uniforme en todas las tablas

**Proyecto:** hospital-belen · **Stage:** 1 (planner) · **Effort:** high
**Fecha:** 2026-07-14
**DISPATCH:** Revisar TODAS las tablas (sin excepciones, por batches). Paginación desde
backend (página + cantidad). Sustituir "mostrar-todo" por paginación real.

---

## Diagnóstico — hoy conviven 3 patrones inconsistentes

| Patrón | Forma | Endpoints |
|--------|-------|-----------|
| **A. Paginado OK** | `{ data, total, page, limit }` | `patients`, `admissions`, `appointments` |
| **B. limit/offset sin total** | `Vec<...>` plano, sin `total` → el front no puede armar páginas | `users` |
| **C. Mostrar-todo** | `Vec` / `{data}` sin límite efectivo | roles, permissions, endpoints, endpoint-groups, menu-items, tenants, doctors, audit-logs, packages, medical-results (labs), inventory (units/products/warehouses/storages/inventory/movements), receipts, admission-extras |

Infra ya existente a reutilizar:
- Backend: `ListOptions{limit,offset,order_bys}` + tope `LIST_MAX_LIMIT`
  (`infrastructure/database/base.rs:229-240`). Falta **count genérico**.
- Frontend: **`usePagination(page,limit)`** (`src/composables/usePagination.ts`) — ya
  hecho, con `total/totalPages/goToPage/setTotal`. Componente `Pagination*` de
  `@inksightdev/ui`. Patrón vivo: `PatientListPage.vue` / `AdmissionListPage.vue`.

**Decisión rectora (effort=high):** unificar en el **patrón A** en TODO endpoint que
alimente una tabla. Un contrato, un envelope, un extractor, un componente. No inventar
por handler.

---

## Contrato canónico (crear primero, luego aplicar por batches)

### Backend — `hospital-belen-api/`

**1. `src/web/dto/pagination.rs` (CREAR)**
```rust
#[derive(Deserialize)]
pub struct PageQuery {
    #[serde(default = "default_page")]  pub page: i64,   // 1-based
    #[serde(default = "default_limit")] pub limit: i64,  // default 25
}
fn default_page() -> i64 { 1 }
fn default_limit() -> i64 { 25 }
impl PageQuery {
    pub fn clamped(&self) -> (i64, i64, i64) {           // (page, limit, offset)
        let page  = self.page.max(1);
        let limit = self.limit.clamp(1, LIST_MAX_LIMIT as i64);
        (page, limit, (page - 1) * limit)
    }
}

#[derive(Serialize)]
pub struct Paginated<T> { pub data: Vec<T>, pub total: i64, pub page: i64, pub limit: i64 }
impl<T> Paginated<T> {
    pub fn new(data: Vec<T>, total: i64, page: i64, limit: i64) -> Self { … }
}
```

**2. Count genérico — `src/infrastructure/database/base.rs` (MODIFICAR)**
Añadir `pub async fn count<MC,F>(db, ctx, filter) -> Result<i64>` que emita
`SELECT COUNT(*)` con el mismo `FilterGroups` que `list` (misma construcción sea-query,
sin limit/offset). Así cada repo paginado obtiene `total` sin SQL crudo por handler.
`// ponytail:` un solo count genérico; nada de un COUNT a mano por entidad como en
`patient.rs:131`.

**3. Alinear los que ya paginan (patrón A)** — `patients`, `admissions`, `appointments`
ya devuelven la forma correcta; solo **verificar** que las llaves sean idénticas
(`data,total,page,limit`) y migrar su COUNT crudo al helper genérico. `appointments`
hoy no devuelve `limit` (`appointment.rs:73`) → añadirlo.

### Frontend — `hospital-belen-web/kairosaid/`

**4. `src/components/common/DataTablePager.vue` (CREAR)** — envoltorio fino sobre
`Pagination*` de `@inksightdev/ui`: props `:page :total :limit`, emite
`@update:page`. Muestra "Página X de N (T registros)". Reemplaza los pagers ad-hoc
copiados en PatientListPage/AdmissionListPage/ReceiptList por uno solo.

**5. Selector de tamaño de página (opcional, ver OPEN QUESTION 2).**

Cada página de tabla: `const pg = usePagination(1, 25)`; en el fetch: `pg.setTotal(res.total)`;
`<DataTablePager :page="pg.currentPage" :total="pg.total" :limit="pg.limit"
@update:page="p => { pg.goToPage(p); fetch() }" />`.

---

## Batches (orden de ejecución sugerido para el coder)

> Regla: **cada batch = backend (handler+repo+count) → servicio front → página**. Un
> batch se cierra compilando (`cargo check` + `npm run build`) antes del siguiente.

### Batch 0 — Contrato base
`dto/pagination.rs` + `base.rs::count` + `DataTablePager.vue` + `usePagination` (ya
existe). Sin cambios de comportamiento; habilita el resto.

### Batch 1 — Admin (el que motivó tickets previos)
| Endpoint (handler) | Front page + service |
|---|---|
| `list_roles` (rbac.rs:72) | RoleListPage.vue / adminService.listRoles |
| `list_permissions` (rbac.rs:30) | PermissionsPage.vue |
| `list_endpoints` (rbac.rs:162) | EndpointListPage.vue |
| `list_endpoint_groups` (endpoint_group.rs:37) | *(inline en Endpoints)* |
| `list_menu_items` (menu_item.rs:40) | MenuItemListPage.vue |
| `list_users` (user.rs:39) — patrón B→A: añadir `total` | UserListPage.vue |
| `list_tenants` (tenant.rs:59) | TenantListPage.vue |
| `list_audit_logs` (audit_log.rs:11) | AuditLogPage.vue |

### Batch 2 — Inventario (mayor volumen real)
| Endpoint | Front |
|---|---|
| `list_products` (inventory.rs:71) | ProductListPage.vue |
| `list_inventory` (inventory.rs:258) | InventoryItemListPage.vue |
| `list_movements` (inventory.rs:334) | InventoryMovementPage.vue |
| `list_warehouses` (inventory.rs:135) | *(si tiene tabla propia; si es select → excluir)* |

### Batch 3 — Clínico / facturación
| Endpoint | Front |
|---|---|
| `list_receipts` (receipt.rs:52) | ReceiptList.vue (ya tiene Pagination parcial → completar) |
| `list_results` (medical_result.rs:36) | LabResultListPage.vue |
| `list_packages` (packages.rs:28) | PackageListPage.vue |
| `list_doctors` (doctor.rs:27) | *(si hay tabla de médicos; hoy DoctorQuery ya tiene page)* |

### Batch 4 — Verificación de los ya-paginados
`patients`, `admissions`, `appointments`: alinear llaves + migrar COUNT al helper +
sustituir su pager ad-hoc por `DataTablePager`. Regresión visual mínima.

---

## Exclusiones explícitas (revisadas, NO son omisiones)

Estos endpoints/tablas **no** llevan paginación servidor; justificación por caso:

- **Feeds de dropdown/autocomplete** (no son tablas): `search_patients`,
  `list_lite_patients` (patient.rs), `list_units`, `list_storages`, `list_all_storages`
  (selects de inventario). Mantener su `limit` pequeño actual.
- **Datos de referencia acotados**: `list_who_tables` (pediatrics) — tabla OMS fija.
- **Sub-tablas de detalle** (colección de UN registro padre, tamaño acotado):
  `PatientDetailPage`, `AdmissionDetailPage` + `list_movements`(admission.rs:304) de esa
  admisión, `PackageDetailPage`, `ReceiptPreview`, `list_extras` (admission_extra),
  `list_growth_points` (por paciente).
- **Agregados / no-lista**: `DashboardPage`, `RevenuePage` (report), `PatientGrowthChartsPage`.
- **`PediatricsGrowthChartsListPage`**: OPEN QUESTION 3 — confirmar si es lista real
  paginable o vista fija.

Si el coder encuentra que alguna "exclusión" sí tiene volumen y tabla propia, paginarla
y anotarlo en `changes.md` — la regla del DISPATCH es *sin excepciones injustificadas*.

---

## OPEN QUESTIONS

1. **`limit` por defecto**: propongo **25** uniforme. ¿Preferís 50 (como users/patients
   hoy) para no cambiar densidad visual? Afecta a todos los batches.
2. **Selector de tamaño de página** (10/25/50) en la UI: ¿lo incluimos ahora o
   hardcodeamos 25? (asumido: hardcode 25, sin selector — YAGNI hasta pedirlo).
3. **`PediatricsGrowthChartsListPage`**: ¿lista paginable o vista fija? (asumido: fija).
4. **Orden por defecto**: para paginar de forma estable hace falta `ORDER BY`
   determinista. ¿`created_at DESC` como default global salvo donde ya exista orden?
   (asumido: sí — sin orden estable, la paginación repite/salta filas).

---

## Edge cases (todos los batches)

- `page` fuera de rango (> totalPages) → devolver `data: []` con `total` real (el front
  corrige a la última página). No error 4xx.
- `limit` > `LIST_MAX_LIMIT` → clamp, no error.
- `total = 0` → `totalPages = 1`, tabla vacía con su empty-state actual (no romper).
- **Filtros + paginación**: el `total` debe reflejar el filtro aplicado (mismo
  `FilterGroups` en `list` y `count`), no el total global. Caso típico: UserListPage con
  filtro de status, PatientListPage con `status != INACTIVE`.
- Cambiar de filtro → resetear a `page = 1` (`pg.reset()` + refetch).
- Concurrencia de páginas: ignorar respuestas obsoletas si el usuario cambia rápido
  (guardar el `page` pedido y descartar si no coincide al resolver). `// ponytail:`
  solo si se observa parpadeo; no lo implementes preventivamente.
- Orden estable (OPEN Q4): sin `ORDER BY` fijo, offset paginación duplica/salta filas.

---

## Patrones existentes a seguir

- Handler paginado de referencia: `admission.rs:64-123` (`AdmissionListQuery` +
  `{data,total,page,limit}`).
- Query struct con defaults serde: `appointment.rs:31-44`.
- Front pager + composable: `PatientListPage.vue:118-197` + `usePagination.ts`.
- `apiService.get(url, params)` ya envía query params (`api.service.ts:43`).

---

## SKILL_RECOMENDADA
Ninguna. Es refactor transversal sobre Axum/sqlx + Vue propios; no hay skill instalable
que pagine este stack. La consistencia la da el contrato canónico, no una dependencia.

---

## Fuera de alcance
- Cursor/keyset pagination (offset simple basta al volumen actual;
  `// ponytail:` migrar a keyset solo si una tabla supera ~100k filas).
- Virtual scrolling / infinite scroll (el DISPATCH pide páginas explícitas).
- Cambiar el componente de tabla; solo se añade el pager y el cableado de datos.
