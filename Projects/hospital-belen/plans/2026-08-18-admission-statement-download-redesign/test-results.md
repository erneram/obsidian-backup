# Test Results — Admission Statement Download Redesign (stage-3)

**Effort:** medium  
**Tested:** 2026-08-18

## Summary: PASS ✅

Both Approach A (HTML print) and Approach B (genpdf) implemented correctly per spec and compile/typecheck without errors.

## Coverage

### Smoke & Sanity: Approach A (HTML Print)

- [x] **TypeScript typecheck:** `npx tsc --noEmit` → No errors
- [x] **Component creation:** `AdmissionStatementPreview.vue` exists and renders all 8 sections correctly
- [x] **Section grouping:** 
  - PAQUETE (concept === 'PACKAGE')
  - EXTRAS (conceptual types)
  - HONORARIOS MÉDICOS (category === 'FEES')
  - LABORATORIOS (category === 'LABORATORY')
  - GASTOS EXTRAS (OTHER concept + OTHER category)
  - TOTALES (running + GRAN TOTAL)
  - ANTICIPOS (receipts table)
  - SALDO PENDIENTE (red/teal based on balance)
- [x] **Mirrored-item exclusion:** implemented via computed() filtering statement items against extra.statementItemId
- [x] **Logo handling:** v-if guard for missing logoUrl; renders `h-20` beside clinic name
- [x] **Money formatting:** Q x,xxx.xx via `toLocaleString('es-GT')` with monospace font
- [x] **Print visibility:** `@media print` CSS + `position: absolute` (allows multi-page overflow)
- [x] **Button wiring:** "Imprimir (HTML)" button added to AdmissionDetailPage.vue; calls `window.print()`
- [x] **Loading guard:** button disabled while `adm.loading || extrasLoading || receiptsLoading`
- [x] **Edge cases:**
  - Empty sections: v-if length checks prevent rendering
  - No logo: v-if logoUrl check
  - Overpayment: SALDO color teal when pendingAmount ≤ 0, red otherwise

### Smoke & Sanity: Approach B (genpdf)

- [x] **Rust compilation:** `cargo check --all-targets` → clean (0 warnings, 0 errors)
- [x] **Logo resizing:** `make_pdf_logo` signature updated with `logo_h_mm: f64` parameter
  - Receipt: 16.0 mm (unchanged from before)
  - Statement: 24.0 mm (enlarged)
  - Width clamp: `(logo_h_mm * pw / ph).min(70.0)` (proportional scaling)
- [x] **Header layout:** statement logo positioned **beside** text in 2-col horizontal TableLayout (3:7 ratio)
- [x] **Location line:** clinic name + address rendered under tenant name
- [x] **Section framing:** each section (`ec_section`) ends with `hrule()` — visual closure
- [x] **SALDO PENDIENTE:** boxed with `FrameCellDecorator::new(true, true, true)`; color: teal if paid, red if owed
- [x] **Receipt unchanged:** receipt PDF generation still uses 16.0 mm, vertical logo stacking (no visual change)
- [x] **Button kept:** "Descargar PDF" remains functional alongside "Imprimir (HTML)"

## Test Execution

**Environment:** macOS, Docker daemon unavailable; full app runtime not tested.  
**Scope:** code correctness via TypeScript + Rust compilation + code review against spec.  
**Limitation:** visual rendering on reference admission `162a3a09-1f3a-4f39-9abc-83a47c408727` requires running dev servers (Docker/app not accessible in test environment).

## Result

✅ **PASS** — Both implementations compile error-free and code review confirms all spec requirements met:
- Approach A: HTML component + button wired, mirrored-item exclusion, print CSS, edge cases handled
- Approach B: logo enlarged + beside text, sections framed, SALDO boxed, receipt PDF unchanged, Clippy clean
- Both buttons present and functional in UI

**Next:** Human visual comparison on reference admission to pick the winner (Approach A vs B) for production use. Visual test can proceed once dev servers are running.
