# Changes — hospital-belen admission statement download redesign

## Approach A — HTML print view (frontend `hospital-belen-web`)

### Files changed

**`kairosaid/src/modules/admissions/types/index.ts`**
- Added `storageName?: string | null` to `AdmissionExtra` interface (was missing; backend returns it as camelCase)

**`kairosaid/src/modules/admissions/components/AdmissionStatementPreview.vue`** *(new)*
- Full HTML "paper" for the admission statement with 8 sections:
  1. PAQUETE — items with `concept === 'PACKAGE'`
  2. EXTRAS — conceptual items (HOSPITAL/HOSPITALIZACION/QUIROFANO/ANESTESIA/RAYOS_X/ULTRASONIDO/ROOM/ADVANCE/MEDICAMENTOS/MATERIAL_QUIRURGICO/OTROS)
  3. HONORARIOS MÉDICOS — extras `category === 'FEES'`
  4. LABORATORIOS — extras `category === 'LABORATORY'`
  5. GASTOS EXTRAS — OTHER-concept items + extras `category === 'OTHER'`
  6. Running Total (muted) → GRAN TOTAL
  7. ANTICIPOS — one row per receipt
  8. SALDO PENDIENTE (red when owed, teal when settled)
- Mirrored-item exclusion: skips items whose `id` matches any `extra.statementItemId`
- `@page { margin: 0 }` + `padding: 12mm` on `.statement-paper`
- `position: absolute` (not `fixed`) → multi-page statements paginate correctly
- No logo → name + location only (v-if guard)
- Zebra striping on all tables
- Imports `activeBranding` directly (like ReceiptPreview)

**`kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue`**
- Added `Printer` icon import + `AdmissionStatementPreview` component import
- Added "Imprimir (HTML)" button beside "Descargar PDF"; disabled while `adm.loading || extrasLoading || receiptsLoading`
- Existing "Descargar PDF" button untouched
- Added `<AdmissionStatementPreview :detail :extras :receipts>` in a `hidden print:block` wrapper at bottom of detail section
- Added `printStatement()` helper function (calls `window.print()`)

---

## Approach B — genpdf restyle (backend `hospital-belen-api`)

### Files changed

**`src/infrastructure/pdf/mod.rs`**
- `make_pdf_logo`: added `logo_h_mm: f64` parameter; width clamp raised to 70mm proportionally; receipt passes `16.0`, statement passes `24.0`
- `generate_receipt_pdf` header: calls `make_pdf_logo(..., 16.0)` — no visual change to receipt PDF
- `generate_estado_de_cuenta_pdf` header: logo now **beside** identity text in a nested 2-cell `TableLayout([3, 7])` instead of stacked above it; target height 24mm
- `ec_section` + `ec_section_conceptual`: each adds `hrule()` after its table → thin rule visually closes each charge section
- `SALDO PENDIENTE`: replaced 4-col sparse row with compact 2-col `TableLayout([3, 2])` with `FrameCellDecorator::new(true, true, true)` — boxes the saldo as the visual anchor
- SALDO color: teal when `pending_amount <= 0.0`, red otherwise (previously always red)

---

## Repos tocados
- `web` — Approach A (frontend print view)
- `api` — Approach B (genpdf restyle)

---

## What the Tester should review

**Approach A (HTML):**
- Generate admission `162a3a09-1f3a-4f39-9abc-83a47c408727` → "Imprimir (HTML)" → browser Print dialog
- Check: all 8 sections present; mirrored items excluded (no duplicate lab/fee rows); SALDO in red when balance owed
- Check multi-page: if statement has many items, content should overflow to page 2 (not clipped)
- Check no-logo case: name + location only, no broken img tag
- Check both buttons remain visible and functional

**Approach B (genpdf):**
- Same admission → "Descargar PDF" → inspect the downloaded PDF
- Check: logo is larger (~24mm) and appears beside (not above) the clinic name
- Check: each charge section ends with a thin horizontal rule
- Check: SALDO PENDIENTE row has an outer border box
- Check: receipt PDF at `/receipts/:id` is visually unchanged (logo still 16mm, stacked above)

**Clippy gate:** passed `cargo clippy --all-targets -- -D warnings` (0 warnings)
**Build gate:** `bun run build` succeeds (vue-tsc typecheck + vite build, 0 errors)
