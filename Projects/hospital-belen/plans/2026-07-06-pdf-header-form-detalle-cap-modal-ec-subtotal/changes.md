# Changes — stage-2 hospital-belen (2026-07-06)

## Files modificados

### `hospital-belen-web/kairosaid/src/modules/receipts/components/ReceiptPreview.vue`
- Agregado `@page { margin: 0 }` + `padding: 10mm; box-sizing: border-box` en print styles
- Elimina el header/footer que Chrome inyecta en `window.print()`

### `hospital-belen-web/kairosaid/src/modules/receipts/components/ReceiptForm.vue`
- Eliminada sección Notes/Observaciones del template
- Eliminada desestructuración `notes`/`notesAttrs` del script
- Concepto extraído del grid → full-width standalone
- Detalle movido al final del form, full-width (antes de Actions)
- Agregada interfaz `pendingAmount?: number` en `Prefill`
- Agregada función `maxForConcept(idx)` que calcula el cap por concepto
- Input Amount: `:max` + clamp `@input` activos cuando `prefill.pendingAmount` existe
- Warning visible si total supera pendingAmount

### `hospital-belen-web/kairosaid/src/modules/receipts/pages/ReceiptCreatePage.vue`
- Agregado `pendingAmount?: number` a la interfaz `Prefill` local
- Agregado `pendingAmount: detail.statement.pendingAmount` al objeto prefill en `onMounted`

### `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue`
- Dialog `editPkgOpen`: `max-w-lg` → `max-w-3xl`

### `hospital-belen-api/src/infrastructure/pdf/mod.rs`
- `ConceptPdfData`: agregado campo `subtotal_label: Option<String>`
- Agregada constante `EC_COLS: [usize; 4] = [2, 7, 3, 2]`
- `ec_section`: signature actualizada a `rows: &[(String, String, Option<String>, f64)]`; body usa 4 columnas; col3 muestra `subtotal_label` en muted 8pt
- `collect` lambda: retorna 4-tuplas con `i.subtotal_label.clone()`
- `hosp.push`: agrega `None` como 3er elemento para extras
- `pay_rows`: tipo cambiado a 4-tupla con `None` en posición 3
- `grand`/`pagado`/`saldo`: usan `EC_COLS` + extra elemento vacío por fila

### `hospital-belen-api/src/web/handlers/admission.rs`
- Mapeo de items: `subtotal_label = Some("qty × Q precio")` si quantity+unit_price presente; descripción queda limpia

## Qué debe revisar el Tester

1. **PDF Talonario (Imprimir)**: Chrome no debe mostrar header con URL ni footer con fecha/página
2. **Form Talonario**: Notes ya no aparece; Concepto es full-width; Detalle aparece al fondo antes de botones
3. **Cap de monto**: Con `prefill.pendingAmount` activo, el input de Amount no puede superar el saldo al hacer tab/blur; warning aparece si total excede
4. **Dialog Editar Paquete**: Debe ser más ancho (max-w-3xl vs max-w-lg anterior)
5. **PDF Estado de Cuenta**: Items con quantity+unit_price muestran "N × Q X.XX" en col3; items sin ellos dejan col3 vacía; subtotales y totales alineados con 4 columnas
