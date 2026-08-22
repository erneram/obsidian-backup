# Changes — stage-2 hospital-belen (2026-07-07)

## Files modificados

### `hospital-belen-api/src/domain/admission/mod.rs`
- `AccountStatementItem`: agregado `storage_name: Option<String>`

### `hospital-belen-api/src/domain/admission_extra/mod.rs`
- `AdmissionExtra`: agregado `storage_name: Option<String>`

### `hospital-belen-api/src/infrastructure/database/repositories/admission_repository.rs`
- Query items: `FROM account_statement_items` → `FROM account_statement_items asi LEFT JOIN storages s`; agrega `s.name AS storage_name` al SELECT

### `hospital-belen-api/src/infrastructure/database/repositories/admission_extra_repository.rs`
- Query `list`: agrega LEFT JOIN storages + `s.name AS storage_name`
- Query `create` RETURNING: agrega `NULL::text AS storage_name`

### `hospital-belen-api/src/infrastructure/pdf/mod.rs`
- `ConceptPdfData`: agregado `storage_name: Option<String>`
- `ExtraPdfLine`: agregado `storage_name: Option<String>`
- `PaymentPdfLine`: eliminado `notes`, agregado `receipt_number: String` + `concepto: Option<String>`
- Agregada constante `EC_SIMPLE_COLS: [usize; 3] = [3, 8, 2]`
- Agregada función `format_date_ddmmyyyy`
- Agregada función `ec_section_conceptual` (3 cols, sin subtotal_label)
- Secciones de cargos en `generate_estado_de_cuenta_pdf`: 4 secciones → 3 secciones:
  - PAQUETE (sin cambio)
  - EXTRAS: todos los conceptos directos via `ec_section_conceptual` (3 cols)
  - EXTRAS DE INVENTARIO: items OTHER + admission_extras via `ec_section` (4 cols, col1=Bodega)
- ANTICIPOS: mapeo usa `receipt_number` + `concepto` + `format_date_ddmmyyyy`

### `hospital-belen-api/src/web/handlers/admission.rs`
- `ExtraPdfLine` mapping: agrega `storage_name: e.storage_name.clone()`
- `ConceptPdfData` mapping: agrega `storage_name: i.storage_name.clone()`
- `payments`: usa `state.receipt_repo.list(...)` con `admission_id_filter=Some(id)` en lugar de movements filtrados por PAYMENT
- Eliminada llamada `list_movements` del handler `download_statement_pdf`

## Qué debe revisar el Tester

1. **PDF EC — secciones**: Solo aparecen PAQUETE, EXTRAS, EXTRAS DE INVENTARIO (no HOSPITALIZACIÓN/EXTRAS, OTROS GASTOS, LABORATORIOS, HONORARIOS por separado)
2. **PDF EC — EXTRAS**: Muestra todos los conceptos directos (HOSPITAL, LABORATORY, FEES, etc.) sin columna subtotal
3. **PDF EC — EXTRAS DE INVENTARIO**: Col1=nombre de bodega (o "—"); col3=qty×precio; incluye tanto items OTHER como admission_extras
4. **PDF EC — ANTICIPOS**: Muestra N° de talonario, concepto del talonario, fecha en DD/MM/YYYY
5. **API**: endpoint `GET /api/admissions/{id}/movements` sigue funcionando (usa `list_movements` en handler propio)
