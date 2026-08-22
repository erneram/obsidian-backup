# changes.md — AdmissionDetailPage: Layout + Columnas + PDF
**project:** hospital-belen  
**dispatch:** stage-2  
**date:** 2026-06-25

---

## Archivos modificados

### `hospital-belen-api/src/domain/admission/mod.rs`
- Agregados `quantity: Option<f64>` y `unit_price: Option<f64>` a `AccountStatementItem`

### `hospital-belen-api/src/infrastructure/database/repositories/admission_repository.rs`
- Query `get()` ahora selecciona `quantity::float8 AS quantity` y `unit_price::float8 AS unit_price`

### `hospital-belen-web/kairosaid/src/modules/admissions/types/index.ts`
- Agregados `quantity?: number` y `unitPrice?: number` a `AccountStatementItem`

### `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue`
**Script:**
- Imports: `FileDown` (lucide), `estadoCuentaService`
- Nuevos computed: `packageItems` (concept=PACKAGE de detail.items), `historyRows` (merge receipts + movements ordenados por fecha)
- `pdfLoading` ref + `downloadPdf()` función (blob download)
- Tipo `HistoryRow` flat para el merge
- Eliminados helpers ya no usados: `MOVEMENT_LABELS`, `movementSign`, `movementAmountClass`, `movementTypeClass`

**Template — nuevo orden de secciones:**
1. Datos Clínicos + Balance de Cuenta — sin cambios
2. Cargos de Inventario — solo extras (sin sub-sección de paquetes)
3. **Paquetes Aplicados** — nueva card `v-if="packageItems.length"` con columnas Desc/Cant./Precio Unit./Total + botón eliminar
4. **Extras de Enfermería** — subida a posición 3 (antes estaba en 4)
5. Add Extra Dialog
6. **Historial de Pagos** — card unificada (reemplaza "Talonarios" + "Historial de Pagos/movements") con `historyRows`
7. Botón "Descargar PDF" en el header

---

## Qué debe revisar el Tester

1. **Backend compilación**: `cargo check` en `hospital-belen-api/` — los nuevos campos deben mapearse sin error sqlx
2. **Paquetes Aplicados**: aplicar un paquete a una admisión y verificar que aparece en la nueva card con cantidad/precio unit (si vienen en DB) o `—`
3. **Cargos de Inventario**: card solo muestra extras (items de inventario), sin paquetes mezclados
4. **Extras de Enfermería**: posición correcta (antes de Historial de Pagos)
5. **Historial de Pagos**: talonarios y movimientos aparecen en la misma tabla ordenados por fecha; solo talonarios muestran botón "Ver"
6. **PDF**: botón "Descargar PDF" dispara descarga del archivo, disabled durante carga
7. **Sin regresiones**: las demás secciones (Datos Clínicos, Balance de Cuenta, dialogs) sin cambios
