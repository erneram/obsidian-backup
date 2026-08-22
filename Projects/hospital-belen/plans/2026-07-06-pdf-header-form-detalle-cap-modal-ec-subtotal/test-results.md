# Test Results — hospital-belen — 5 fixes: PDF header, form Detalle, amount cap, modal size, EC PDF subtotal

**Ejecutado:** 2026-07-06  
**Modo:** stage-3 effort=medium  
**Resultado:** ✅ PASS

---

## Verificaciones

### 1. `cargo build` ✅
- `Finished dev profile` — exit 0, 70 warnings pre-existentes, 0 errores

### 2. `tsc --noEmit` ✅
- TypeScript: No errors found

### 3. Fix 1 — `@page { margin: 0 }` en ReceiptPreview.vue ✅
- Línea 185: `@page { margin: 0 }` ✅
- Líneas 201-202: `padding: 10mm; box-sizing: border-box` ✅

### 4. Fix 2/3 — ReceiptForm.vue ✅
- `notesAttrs` eliminado — no existe en el archivo ✅
- Sección "Notes/Observaciones" eliminada del template ✅ (`notes` solo persiste como campo interno del form values)
- `maxForConcept(idx)` presente (línea 278) ✅
- `:max="maxForConcept(idx)"` en Amount input (línea 134) ✅
- Warning si total > pendingAmount (línea 163) ✅
- `pendingAmount?: number` en Prefill interface (línea 223) ✅

### 5. Fix 3 — ReceiptCreatePage.vue ✅
- `pendingAmount: detail.statement.pendingAmount` en prefill (línea 96 + 103) ✅

### 6. Fix 5 — PDF Estado de Cuenta (pdf/mod.rs + admission.rs) ✅
- `EC_COLS: [usize; 4] = [2, 7, 3, 2]` (línea 41) ✅
- `subtotal_label: Option<String>` en `ConceptPdfData` (línea 62) ✅
- `ec_section` firma 4-tupla (línea 458) ✅
- `subtotal_label` en mapeo de items en `admission.rs` (líneas 322, 330) ✅

### 7. Fix 4 — Dialog Editar Paquete ✅
- `AdmissionDetailPage.vue` línea 521: `<DialogContent class="max-w-3xl">` ✅
