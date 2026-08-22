# Spec — admissions: fix 500 en items, Extras invisibles, date filters UI

Proyecto: hospital-belen · effort=high · 3 issues (2 root causes reales, 1 UI).

---

## Issue 1 — POST /api/admissions/{id}/items → 500 (ROOT CAUSE ENCONTRADO)

**Causa:** `AccountStatementItem` (struct `sqlx::FromRow`) tiene el campo
`storage_name: Option<String>` — `src/domain/admission/mod.rs:57`.
sqlx `FromRow` **exige columna con ese nombre en el result set** (aunque sea
`Option`; columna ausente = error `ColumnNotFound`, NO se trata como NULL).

Dos `query_as::<_, AccountStatementItem>` con `INSERT ... RETURNING` **omiten**
`storage_name` en el RETURNING → `ColumnNotFound` → `DomainError::Database` → 500:

- `add_item` — `src/infrastructure/database/repositories/admission_repository.rs`
  RETURNING líneas ~377-385 (falta `storage_name`).
- `apply_package` — mismo archivo, RETURNING líneas ~519-527 (también falta).

El `get()` (línea ~202) sí lo trae vía `LEFT JOIN storages s ... s.name AS storage_name`.
Fue introducido con el JOIN y no se propagó a los RETURNING → regresión.

**Fix:** en ambos RETURNING agregar una columna constante:
```sql
    unit_price::float8      AS unit_price,
    NULL::text              AS storage_name,
    created_at::text        AS created_at
```
(los items creados por estos paths no tienen storage asociado → `NULL` correcto.)

**Verificación obligatoria (coder):** grep en el repo
`grep -rn "query_as::<_, AccountStatementItem>" src/` y confirmar que **cada**
sitio (add_item, apply_package, replace_inventory_items ~L863, get ~L192)
devuelve `storage_name`. Cualquiera que falte tiene el mismo bug latente.

**Edge cases:**
- `NULL::text` debe castearse explícito o sqlx puede inferir tipo desconocido.
- No tocar la firma pública del struct ni el JSON de salida (serde `storageName`).

---

## Issue 2 — "no se ven Extras" en estado de cuenta

**Causa primaria:** encadenada al Issue 1. El diálogo "Agregar Extra"
(`AdmissionDetailPage.vue:775 submitAddExtra`) llama `adm.addItem` →
POST /items → 500 → nada se guarda. **Arreglar Issue 1 resuelve el síntoma
principal.** Coder debe re-testear extras tras el fix del backend.

**Bug de display secundario (real, arreglar también):**
`AdmissionDetailPage.vue:907-911` — `conceptualItems` filtra excluyendo
`concept === 'OTROS'`:
```ts
i => i.concept !== 'PACKAGE' && i.concept !== 'OTROS' && i.concept !== 'OTHER'
```
Pero el `<Select>` del diálogo usa `CONCEPT_OPTIONS` que **incluye `'OTROS'`**
(`types/index.ts`). Un extra guardado con concepto OTROS queda excluido de la
tabla Extras Y de `inventoryChargeItems` (que solo matchea `'OTHER'`, inglés,
reservado a cargos de inventario) → desaparece.

**Fix:** en `conceptualItems`, quitar la exclusión de `'OTROS'`:
```ts
const conceptualItems = computed(
  () => detail.value?.items.filter(
    i => i.concept !== 'PACKAGE' && i.concept !== 'OTHER'
  ) ?? []
)
```
Distinción a respetar: `OTHER` (inglés) = cargo de inventario → tabla "Cargos de
Inventario". `OTROS` (español) = extra manual → tabla "Extras". No mezclar.

---

## Issue 3 — Date filters "feos" → componentes @inksightdev/ui

**Ubicación:** `AdmissionListPage.vue:32-46` — dos `<Input type="date">` con
clases de borde manuales (inconsistente con el resto del UI).

**Fix:** reemplazar por `DatePicker` de `@inksightdev/ui` (ya se importa desde
ese paquete en el archivo; añadir `DatePicker` a la lista de imports L177-189).

**Contrato de DatePicker** (de `@inksightdev/ui` d.ts):
- `v-model` es un **`Date`** (no string). Emite `update:modelValue: (date: Date)`.
- Props: `placeholder?: string`, `locale?: string` (usar `'es-GT'`),
  `minDate?: Date`, `maxDate?: Date`, `disabled?`, `class?`.
- Patrón de referencia ya en repo: `src/modules/pediatrics/components/GrowthPointForm.vue:6` (`v-model="measuredAtDate"`).

**Cambios de estado (L197-230):**
- `dateFrom`/`dateTo` hoy son `ref('')` (string). Cambiar a `ref<Date>()`
  (`Date | undefined`).
- El backend espera `dateFrom`/`dateTo` como **string** `Option<String>`
  (`handlers/admission.rs:56-59`, filtra sobre `admitted_at`). → convertir al
  construir params en `load()`:
  ```ts
  function toYmd(d?: Date) {
    if (!d) return undefined
    // fecha local, NO toISOString (evita corrimiento por timezone)
    return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`
  }
  ```
  y usar `dateFrom: toYmd(dateFrom.value)`, `dateTo: toYmd(dateTo.value)`.
- El `@change="load(1)"` de los `<Input>` no aplica: usar
  `@update:modelValue="load(1)"` en cada DatePicker (o un `watch([dateFrom,dateTo])`).
- `clearFilters` (L229-230): setear a `undefined`, no `''`.

**Edge cases:**
- `placeholder` para cada uno: "Desde" / "Hasta".
- Mantener el separador "—" y el layout `flex gap-2`.
- No romper los otros filtros (patientSearch, statusFilter).

---

## Diseño visual

- DatePicker usa el estilo nativo del design system → hereda tokens (bg-background,
  border, radius) automáticamente. No añadir clases de borde manuales.
- Layout: `<div class="flex items-center gap-2">` con `<DatePicker placeholder="Desde" />`
  `<span class="text-muted-foreground text-xs">—</span>` `<DatePicker placeholder="Hasta" />`.
- `locale="es-GT"` para formato de fecha guatemalteco.

---

## Archivos a modificar

Backend (Rust):
- `hospital-belen-api/src/infrastructure/database/repositories/admission_repository.rs`
  — RETURNING de `add_item` (~L385) y `apply_package` (~L527): + `NULL::text AS storage_name`.

Frontend (Vue):
- `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue`
  — `conceptualItems` filter (L907-911).
- `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionListPage.vue`
  — import DatePicker, refs de fecha a Date, helper toYmd, markup L32-46, clearFilters.

---

## Testing

Backend:
- `cargo test` + smoke manual: `POST /api/admissions/{id}/items`
  body `{"concept":"MEDICAMENTOS","description":"x","amount":10}` → 201, no 500.
- Repetir con concepto `OTROS` → 201.
- `POST /api/admissions/{id}/packages` → sigue 201 (no romper apply_package).

Frontend (build/typecheck):
- `bun run build` o `vue-tsc --noEmit` — DatePicker tipado con Date, sin errores.
- Manual: agregar Extra concepto MEDICAMENTOS → aparece en tabla Extras.
- Agregar Extra concepto OTROS → aparece en Extras (no desaparece).
- Filtrar por rango de fechas → params `dateFrom`/`dateTo` en formato YYYY-MM-DD.

---

## Nota ops (no bloqueante, flag para revisión)

`migrations/028_extend_stmt_concept.sql` usa `ALTER TYPE ... ADD VALUE` y el
migrator (`migrator.rs:apply`) envuelve cada archivo en transacción. En PG12+
`ADD VALUE` corre en tx **si el valor no se usa en la misma tx** — 028 solo
agrega, no usa → OK. Pero si la DB desplegada es PG<12 o el runner cambia,
028 fallaría y el enum no tendría los valores español → 500 al insertar.
No parte de este ciclo; verificar que `stmt_concept` en la DB de prod tenga
HOSPITALIZACION/MEDICAMENTOS/OTROS/etc. antes de cerrar.
