# Test Results — hospital-belen — EC PDF: extras unificados, bodega, anticipos

**Ejecutado:** 2026-07-08  
**Modo:** stage-3 effort=medium  
**Resultado:** ✅ PASS

---

## Verificaciones

### 1. `cargo build` ✅
- `Finished dev profile` — exit 0, 70 warnings pre-existentes, 0 errores

### 2. `tsc --noEmit` ✅
- TypeScript: No errors found

### 3. `storage_name` en structs de dominio ✅
- `admission/mod.rs` línea 57: `pub storage_name: Option<String>` ✅
- `admission_extra/mod.rs` línea 16: `pub storage_name: Option<String>` ✅

### 4. PDF — estructura de secciones ✅
- `ec_section("PAQUETE", ...)` (l.742) — 4-col, concept=PACKAGE ✅
- `ec_section_conceptual("EXTRAS", &conceptual_rows)` (l.759) — 3-col, todos los conceptos directos ✅
- `ec_section("EXTRAS DE INVENTARIO", &inv_rows)` (l.772) — 4-col, col1=bodega, items OTHER + admission_extras ✅
- `ec_section("ANTICIPOS", &pay_rows)` (l.811) — receipt_number + concepto + fecha DD/MM/YYYY ✅

Secciones anteriores eliminadas: HOSPITALIZACIÓN/EXTRAS, OTROS GASTOS, LABORATORIOS, HONORARIOS MÉDICOS — consolidados en `CONCEPTUAL_TYPES` array (l.745-753).

### 5. PDF — helpers nuevos ✅
- `EC_SIMPLE_COLS: [usize; 3] = [3, 8, 2]` (l.43) ✅
- `ec_section_conceptual` (l.551) ✅
- `format_date_ddmmyyyy` (l.147) ✅
- `PaymentPdfLine` con `receipt_number: String` + `concepto: Option<String>` (l.446-448) ✅

### 6. EXTRAS DE INVENTARIO — bodega fallback ✅
- items OTHER: `i.storage_name.unwrap_or("—")` (l.764) ✅
- admission_extras: `e.storage_name.unwrap_or("—")` (l.768) ✅
