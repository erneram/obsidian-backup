# Spec — hospital-belen — 4 issues batch

**Generado:** 2026-06-26  
**Modo:** stage-1

---

## Contexto de la base de datos y arq.

- API en Rust (Axum), BD PostgreSQL. Migraciones en `hospital-belen-api/migrations/` corren en orden al arrancar.
- Multi-tenant: cada dato tiene `tenant_id`. El dev-seed (`src/infrastructure/dev_seed.rs`) crea los tenants de prueba al arranque.
- Enum de base de datos relevante: `stmt_concept ∈ {HOSPITAL, LABORATORY, FEES, ADVANCE, ROOM, PACKAGE, OTHER}` — sólo inglés.

---

## Issue 1 — Error enum DB: `HOSPITALIZACION` y `OTROS` inválidos

### Diagnóstico

`stmt_concept` en DB sólo tiene valores en inglés. El frontend (`admissions/types/index.ts`) envía valores en español:
`HOSPITALIZACION, LABORATORIO, HONORARIOS_MEDICOS, MEDICAMENTOS, QUIROFANO, ANESTESIA, MATERIAL_QUIRURGICO, RAYOS_X, ULTRASONIDO, OTROS`.
Postgres rechaza el cast `$3::stmt_concept` cuando el valor no existe en el enum.

Dos puntos de fallo:
1. `admission_repository.rs:372` — bind user-supplied concept → cast falla con valores españoles
2. `admission_extra_repository.rs:93` — hardcode `'OTROS'::stmt_concept` → `OTROS` no existe en el enum

### Archivos a crear/modificar

| Archivo | Acción |
|---|---|
| `hospital-belen-api/migrations/028_extend_stmt_concept.sql` | **Crear** |
| `hospital-belen-api/src/infrastructure/database/repositories/admission_extra_repository.rs` | Verificar (el hardcode `'OTROS'` quedará válido tras la migración, no requiere cambio de código) |

### Contenido de `028_extend_stmt_concept.sql`

```sql
-- 028 — Extiende stmt_concept con los valores en español que usa el frontend.
-- Los valores en inglés existentes (HOSPITAL, LABORATORY, etc.) se conservan.
ALTER TYPE stmt_concept ADD VALUE IF NOT EXISTS 'HOSPITALIZACION';
ALTER TYPE stmt_concept ADD VALUE IF NOT EXISTS 'LABORATORIO';
ALTER TYPE stmt_concept ADD VALUE IF NOT EXISTS 'HONORARIOS_MEDICOS';
ALTER TYPE stmt_concept ADD VALUE IF NOT EXISTS 'MEDICAMENTOS';
ALTER TYPE stmt_concept ADD VALUE IF NOT EXISTS 'QUIROFANO';
ALTER TYPE stmt_concept ADD VALUE IF NOT EXISTS 'ANESTESIA';
ALTER TYPE stmt_concept ADD VALUE IF NOT EXISTS 'MATERIAL_QUIRURGICO';
ALTER TYPE stmt_concept ADD VALUE IF NOT EXISTS 'RAYOS_X';
ALTER TYPE stmt_concept ADD VALUE IF NOT EXISTS 'ULTRASONIDO';
ALTER TYPE stmt_concept ADD VALUE IF NOT EXISTS 'OTROS';
```

`ADD VALUE IF NOT EXISTS` es idempotente en PostgreSQL.

### Edge cases
- `admission_extra_repository.rs:93` usa `'OTROS'::stmt_concept` — quedará válido tras migración. No requiere cambio de código.
- El PDF en `src/infrastructure/pdf/mod.rs` usa `"OTROS GASTOS"` como string de display, no como valor enum — sin impacto.
- `receipts.concepto stmt_concept` (mig 027) también acepta los nuevos valores automáticamente.

---

## Issue 2 — Paquetes médicos no aparecen en la web

### Diagnóstico

**Root cause: mismatch de tenant_id entre seed de paquetes y el tenant real del login.**

La migración `003_seed_surgery_packages.sql` siembra `medical_packages`, `warehouses`, `storages`, `inventory_items` con `tenant_id = '10000000-0000-0000-0000-000000000001'` (el tenant creado en `002_seed_rbac.sql` con slug `'default'`).

El dev-seed (`dev_seed.rs`) llama `ensure_demo_tenant("hospital-belen", ...)` → busca `SELECT id FROM tenants WHERE slug = 'hospital-belen'` → no encuentra nada (el slug del tenant existente es `'default'`) → **crea un NUEVO tenant con UUID aleatorio**. Los usuarios (`admin@hospitalbelen.com` etc.) quedan en ese nuevo UUID. Al hacer `GET /api/packages` el API filtra `WHERE tenant_id = <nuevo_uuid>` → lista vacía.

### Fix

**Nueva migración** que renombra el slug del tenant fundacional:

| Archivo | Acción |
|---|---|
| `hospital-belen-api/migrations/029_rename_default_tenant.sql` | **Crear** |

```sql
-- 029 — Alinea el slug del tenant fundacional con lo que espera dev_seed.
-- dev_seed busca slug='hospital-belen'; la migración 002 lo creó como 'default'.
UPDATE tenants
   SET slug = 'hospital-belen',
       name = 'Hospital Belén'
 WHERE id   = '10000000-0000-0000-0000-000000000001'
   AND slug = 'default';
```

Tras esta migración:
- `ensure_demo_tenant("hospital-belen")` encuentra el tenant por slug y retorna `'10000000-0000-0000-0000-000000000001'`
- Todos los paquetes, warehouses, productos e items sembrados en `003` quedan inmediatamente visibles para los usuarios de Hospital Belén
- Si existe un tenant aleatorio previo de desarrollo, sus datos quedan huérfanos (dev DB, aceptable; se resuelve bajando el volumen Docker)

### Edge cases
- Si la DB ya tiene slug='hospital-belen' en otro tenant, el `WHERE slug = 'default'` protege la actualización (no sobrescribe)
- Migración idempotente: si el slug ya es 'hospital-belen', el UPDATE afecta 0 filas

### Conexión seed ↔ UI
Después de la migración, el flujo es:
1. Paquetes aparecen en `/packages` (lista completa 116 paquetes de cirugía)
2. Al crear una admisión, `POST /api/admissions/{id}/packages` aplica el paquete → inserta `account_statement_items` con `concept='PACKAGE'`

---

## Issue 3 — UI Admisiones: filtros en columna, deben ir en fila horizontal

### Diagnóstico

`AdmissionListPage.vue` línea 22: `<SelectTrigger class="w-full">` dentro de un `flex flex-wrap`. La clase `w-full` fuerza al Select de estado a ocupar todo el ancho del contenedor, rompiendo la fila y apilando los elementos en columna.

### Archivos a modificar

| Archivo | Cambio |
|---|---|
| `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionListPage.vue` | Reemplazar `w-full` en SelectTrigger |

### Cambio exacto

**Antes (línea 22):**
```html
<SelectTrigger class="w-full">
```

**Después:**
```html
<SelectTrigger class="w-48">
```

`w-48` (192px) es suficiente para los labels de estado más largos ("Parcialmente Pagado").

La fila resultante queda: `[Input paciente (flex-1, max 448px)] [Select estado (192px)] [Input fecha-desde] [—] [Input fecha-hasta] [Limpiar]` — todo en una sola línea en pantallas ≥ 1024px, con wrap en móvil (comportamiento aceptable).

---

## Issue 4 — Bodegas de testing para inventario

### Diagnóstico

Después del fix del Issue 2, el tenant `'10000000-0000-0000-0000-000000000001'` ya tiene:
- Warehouse `SALA-OP` → Storage `OR` con 105 insumos quirúrgicos
- Warehouse `SERV` → Storage `SERVICIOS` con 7 servicios
- Warehouse `ENF` → Storage `ENF-GEN` (vacío)
- Warehouse `PISOS` → Storage `PISOS-GEN` (vacío)

Para testing de inventario se necesitan bodegas adicionales con items asignados que se puedan probar en la UI sin tocar datos operativos.

### Archivos a modificar

| Archivo | Cambio |
|---|---|
| `hospital-belen-api/src/infrastructure/dev_seed.rs` | Añadir `ensure_test_warehouses()` llamada desde `run()` |

### Firma

```rust
async fn ensure_test_warehouses(db: &Db, tenant_id: Uuid, seeder_id: Uuid) -> Result<()>
```

### Lógica (idempotente con `ON CONFLICT DO NOTHING`)

1. **Warehouses** — insertar dos bodegas de prueba:
   - code `TEST-A`, name `"Bodega de Prueba A"`
   - code `TEST-B`, name `"Bodega de Prueba B"`
   - `ON CONFLICT (tenant_id, code) DO NOTHING`
   - `created_by = seeder_id`

2. **Storages** — un storage por bodega:
   - `GEN-A` en TEST-A, `GEN-B` en TEST-B
   - `ON CONFLICT (tenant_id, warehouse_id, code) DO NOTHING`

3. **Inventory items** — asignar 5 productos del catálogo a cada storage:
   ```sql
   INSERT INTO inventory_items (tenant_id, storage_id, product_id, quantity, min_quantity, unit_cost)
   SELECT $1, $2, p.id, 100, 10, p.default_price
   FROM products WHERE tenant_id = $1 LIMIT 5
   ON CONFLICT (tenant_id, storage_id, product_id) DO NOTHING
   ```

### Llamada en `run()`

```rust
ensure_test_warehouses(db, belen_id, seeder_id).await?;
```

Insertar después de `ensure_dev_appointments(...)`.

### Patrón existente a seguir

`ensure_patients` / `ensure_dev_patients` en el mismo archivo — misma estructura.

### Edge cases
- Si no existen productos en el catálogo aún, los inventory_items no se insertan (subquery retorna vacío). En el flujo normal las migraciones 003 ya los crearon.
- `seeder_id` proviene del valor de retorno de `ensure_admin` al inicio de `run()`.

---

## Orden de implementación sugerido

1. `migrations/028_extend_stmt_concept.sql` — fix crítico de enum
2. `migrations/029_rename_default_tenant.sql` — fix de tenant, debloquea paquetes y warehouses
3. `AdmissionListPage.vue` — cambio de clase CSS (w-full → w-48)
4. `dev_seed.rs` — ensure_test_warehouses

---

## Patrones a seguir

- Migraciones: `hospital-belen-api/migrations/001_schema.sql` (DDL), `002_seed_rbac.sql` (seed idempotente)
- dev_seed: `src/infrastructure/dev_seed.rs` — `ensure_*` functions, `ON CONFLICT DO NOTHING`
- sqlx: `sqlx::query_as::<_, T>(r#"..."#).bind(...).fetch_one/all(db/tx)`
