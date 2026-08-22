# changes.md — Search fix, Historial, Editar paquete
**project:** hospital-belen  
**dispatch:** stage-2 (effort=high)  
**date:** 2026-07-05

---

## Archivos modificados

### `hospital-belen-web/kairosaid/src/modules/estado-cuenta/components/StepItems.vue`
- **F1**: `:value="p.id"` → `:value="p.name"` (package CommandItem)
- **F1**: `:value="s.id"` → `:value="s.name"` (storage CommandItem)
- **F1**: `:value="item.id"` → `:value="item.productName"` (inventory item CommandItem)

### `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue`
- **F1**: `:value="s.id"` → `:value="s.name"` (bodega CommandItem en modal inventario)
- **F1**: `:value="item.id"` → `:value="item.productName"` (ítem CommandItem en modal inventario)
- **F1**: `:value="p.id"` → `:value="p.name"` (package CommandItem en modal Apply Package)
- **F2**: `historyRows` — `notes` usa `r.detalle` (truncado 60 chars) con fallback a `r.concepto`; `by` usa `r.createdByName`
- **F3**: botón "Aplicar Paquete" eliminado del header
- **F3**: botón "Editar" en CardHeader de "Paquetes Aplicados" (visible si hay items)
- **F3**: dialog "Editar cargos" con lista editable (stepper + toggle delete), `openEditPackage()` + `submitEditPackage()`

### `hospital-belen-web/kairosaid/src/modules/receipts/types/index.ts`
- **F2**: `createdByName?: string | null` agregado a `Receipt`

### `hospital-belen-api/src/domain/receipt/mod.rs`
- **F2**: `created_by_name: Option<String>` en `ReceiptRow` y `Receipt` (con `#[serde(rename = "createdByName")]`)

### `hospital-belen-api/src/infrastructure/database/repositories/receipt_repository.rs`
- **F2**: `LEFT JOIN users u ON u.id = r.created_by` en queries list + get
- **F2**: `(u.first_name || ' ' || u.last_name) AS created_by_name` en SELECT de ambas queries
- **F2**: `created_by_name: row.created_by_name` en `row_to_receipt()`

### `hospital-belen-api/src/infrastructure/pdf/mod.rs`
- **F2**: `notes: r.detalle.clone()` (antes era `None`) en `From<&Receipt>`
- **F2**: bloque "Detalle: {notes}" movido a después del nombre del paciente, antes del hrule
- **F2**: bloque "NOTES" después del TOTAL eliminado

---

## Qué debe revisar el Tester

1. **Search F1**: escribir en combobox de bodega/paquete/item → items se filtran por nombre visible
2. **F2 Historial**: columna "Notas" muestra el texto del campo `detalle` del talonario; columna "Por" muestra nombre del creador
3. **F2 PDF talonario**: campo "Detalle:" aparece bajo el nombre del paciente antes de la tabla de conceptos
4. **F3 Editar**: botón "Editar" visible cuando hay package/inventory items; dialog permite cambiar cantidades y marcar para eliminar; guardar actualiza la vista
5. **F3 No regresión**: "Agregar cargo" sigue funcionando independiente
