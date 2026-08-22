# Test Results — hospital-belen — AdmissionDetailPage: Layout + Columnas + PDF

**Ejecutado:** 2026-07-02  
**Modo:** stage-3 effort=high  
**Resultado:** ✅ PASS (con hallazgo de tipo)

---

## Verificaciones

### 1. Backend — `cargo build` ✅
- Compilación limpia, exit code 0
- Campos `quantity: Option<f64>` y `unit_price: Option<f64>` en `AccountStatementItem` — mapeados correctamente por sqlx

### 2. Tests Rust — N/A
- No existen unit tests para `AccountStatementItem` ni `admission_repository` en el codebase
- No hay `#[cfg(test)]` en los archivos de admission

### 3. Frontend — TypeScript `tsc --noEmit` ✅
- **TypeScript: No errors found**
- `packageItems`, `historyRows`, `downloadPdf`, `pdfLoading` — todos tipados correctamente

### 4. HistoryRow — hallazgo ⚠️
**Spec pide** discriminated union:
```ts
type HistoryRow =
  | { kind: 'receipt'; id: string; number: string; ... }
  | { kind: 'movement'; id: string; movementType: string; ... }
```

**Implementación** usa flat type:
```ts
type HistoryRow = {
  kind: 'receipt' | 'movement'
  id: string
  number: string | null   // presente en todas las filas
  movementType: string | null  // presente en todas las filas
  ...
}
```

**Impacto:** TypeScript no estrecha el tipo al hacer `if (row.kind === 'receipt')` — `row.number` sigue siendo `string | null` en lugar de `string`. No causa errores de compilación (tsc limpio) ni errores runtime (template usa ternarios `row.kind === 'receipt' ? row.number : '—'`). Es una desviación de spec sin efecto funcional.

---

## Archivos verificados

| Archivo | Estado |
|---|---|
| `hospital-belen-api/src/domain/admission/mod.rs` — `quantity`, `unit_price` | ✅ |
| `hospital-belen-api/src/infrastructure/database/repositories/admission_repository.rs` — query actualizada | ✅ |
| `hospital-belen-web/kairosaid/src/modules/admissions/types/index.ts` — `quantity?`, `unitPrice?` | ✅ |
| `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue` | ✅ |

---

## Resumen

- cargo build: ✅ limpio
- TypeScript: ✅ sin errores  
- Rust tests: N/A (no existen)
- HistoryRow: ⚠️ flat type vs discriminated union del spec — sin errores TS, sin impacto funcional
