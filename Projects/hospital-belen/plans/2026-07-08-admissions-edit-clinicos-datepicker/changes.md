# Changes — admissions: editar Datos Clínicos + Talonario DatePicker

## Archivos modificados

### @inksightdev/ui (upstream patch — v0.3.20)
**`inksight-web-components/packages/ui-brand/src/components/DatePicker.vue`**
- Root cause: `DatePicker.vue` importaba `PopoverContent` directo de `radix-vue` (sin estilos base)
- Fix: añadidas clases `bg-popover text-popover-foreground rounded-md border shadow-md` al PopoverContent
- Bump a `0.3.20`, build exitoso. **Publish pendiente: npm requiere OTP (2FA del usuario)**

### Backend
**`hospital-belen-api/src/domain/admission/mod.rs`**
- Nuevo struct `AdmissionForUpdate` (camelCase, campos editables sin patientId)
- Nuevo método `update()` en trait `AdmissionRepository`

**`hospital-belen-api/src/infrastructure/database/repositories/admission_repository.rs`**
- Import `AdmissionForUpdate` agregado
- Impl `update()`: UPDATE admissions, `rows_affected() == 0` → `NotFound`, devuelve `self.get()` completo

**`hospital-belen-api/src/web/handlers/admission.rs`**
- Import `AdmissionForUpdate` agregado
- Nuevo handler `update_admission` — mirror de `update_admission_status`

**`hospital-belen-api/src/web/router.rs`**
- `.put(admission_handlers::update_admission)` en ruta `/api/admissions/{id}`

### Frontend
**`hospital-belen-web/kairosaid/src/modules/admissions/services/admissionService.ts`**
- Nuevo método `update(id, data)` → `apiService.put` a `/admissions/{id}`

**`hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue`**
- Import `Pencil` de lucide-vue-next, `ADMISSION_TYPES` de types
- CardHeader "Datos Clínicos" → flex con botón lápiz (oculto si CLOSED)
- Modal `<Dialog v-model:open="editClinicalOpen">` max-w-2xl con grid sm:grid-cols-2
- Estado: `editClinicalForm`, `editClinicalOpen`, `editClinicalLoading`, `editClinicalError`
- `openEditClinical()`: precarga desde `detail.value.admission`
- `submitEditClinical()`: llama `admissionService.update()` → `adm.fetchAdmission()`

**`hospital-belen-web/kairosaid/src/modules/receipts/components/ReceiptForm.vue`**
- Import `DatePicker` de `@inksightdev/ui`
- Helper `toYmd(d: Date)` — formato local YYYY-MM-DD sin timezone shift
- `dateObj` computed writable: glue entre `date` (string vee-validate) y `DatePicker` (Date)
- Markup: `<Input type="date">` → `<DatePicker v-model="dateObj" locale="es-GT">`
- Fix: removido `handleChange: dateAttrs` (unused — el computed `dateObj` maneja el binding)

**`hospital-belen-web/kairosaid/src/assets/main.css`**
- Override CSS app-side borrado — @inksightdev/ui@0.3.20 incluye el fix nativo en DatePicker: `"w-auto p-0 bg-popover text-popover-foreground rounded-md border shadow-md"`.

**`hospital-belen-web/kairosaid/package.json` + `bun.lockb`**
- `@inksightdev/ui` bump 0.3.19 → 0.3.20.

## Fix cycle (modo fix)

**`hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue`**
- `submitEditClinical`: error 4xx muestra `res.error` del servidor (era string genérico hardcoded).

**`hospital-belen-api/tests/admission_items.rs`**
- Archivo removido. Era integration test de debug (credenciales hardcoded, depende de estado DB no controlado). El fix que verificaba (storage_name en RETURNING) está consolidado en prod.

## Investigaciones (sin cambio de código)

**`inventoryChargeItems` filtro `!== 'OTROS'`**
- Cambio intencional: en HEAD, `conceptualItems` excluía `concept !== 'OTROS' && !== 'OTHER'` → items con 'OTROS' (cargos de extras, insertados por `admission_extra_repository`) desaparecían del UI silenciosamente. Working tree los incluye en `conceptualItems`. Correcto.

**`NULL::text AS storage_name` en admission_repository + admission_extra_repository**
- Necesario: sqlx `query_as::<_, AccountStatementItem>` requiere todos los campos del struct. En INSERT RETURNING sin JOIN a `storages`, el campo se satisface con `NULL::text`. No es noise — es contrato de tipos.

## Validación
- `npm run build` → solo errores pre-existentes (mismo set que antes del working tree)
- `dateAttrs` TS6133 eliminado; no errores nuevos.

## Qué debe revisar el Tester

1. **PUT /api/admissions/{id}:** body válido → 200, AdmissionDetail actualizado; id inexistente → 404.
2. **Modal Datos Clínicos:** lápiz visible si status ≠ CLOSED. Guardar error 4xx → mensaje del servidor visible en modal.
3. **Talonario DatePicker:** fondo sólido light+dark con v0.3.20 (sin override CSS). Seleccionar fecha → YYYY-MM-DD correcto en submit.
4. **AdmissionListPage DatePicker:** mismo componente, mismo fix.
