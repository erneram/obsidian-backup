# Spec — Redesign admission "Estado de Cuenta" download

**Project:** hospital-belen · **Effort:** medium · **Scope:** frontend (`hospital-belen-web`) + backend (`hospital-belen-api`)

## Directive — build BOTH, then compare
Per human feedback: implement **both** redesign approaches side by side, keep both entry points
wired so they can be generated from the same admission and compared visually, then pick the winner.
- **Approach A — HTML print view** (frontend). True invoice styling, zero deps.
- **Approach B — server `genpdf` restyle** (backend). Same PDF pipeline, better-looking output within
  genpdf's limits.
Do NOT delete either during this task. The losing approach is removed in a follow-up only after the
human picks — see **Comparison & Decision** at the bottom.

## Context (current implementation)
- Button "Descargar PDF" in `AdmissionDetailPage.vue` (line ~26) calls `downloadPdf()` (~line 1215),
  which hits `GET /admissions/:id/pdf` → Rust `genpdf` (`api/src/infrastructure/pdf/mod.rs`,
  `generate_estado_de_cuenta_pdf`). Output looks like a raw data table (genpdf can't fill cells or
  zebra-stripe). Approach A replaces it in the UI; Approach B restyles it in place.
- **Existing pattern to mirror:** `modules/receipts/components/ReceiptPreview.vue` — an HTML "paper"
  styled with Tailwind + a `@media print` visibility toggle + `window.print()`. Reuse its structure,
  print CSS, and visual language wholesale.
- **Branding source:** `@/composables/useBranding` → `activeBranding` (`{ name, logoUrl, ... }`).
  `TenantBranding` has NO `address` — location is hardcoded `"Guatemala, Guatemala"` in
  ReceiptPreview; do the same here. // ponytail: add `address` to branding endpoint later if a
  tenant needs a real location.
- **All data already on the page** — the print component must REUSE it, not refetch:
  `detail` (`computed`, has `admission`, `statement`, `items[]`, `patientName`),
  `extras` (`ref<AdmissionExtra[]>`), `receipts` (fetched list), plus existing grouping computeds
  (`packageItems`, `inventoryChargeItems`, `otherExtras`). Coder: read the `<script>` of
  `AdmissionDetailPage.vue` and lift these into the print component's props/computeds.

# Approach A — HTML print view (frontend)

## Files
### Create
- `modules/admissions/components/AdmissionStatementPreview.vue`
  - Props: `detail: AdmissionDetail`, `extras: AdmissionExtra[]`, `receipts: Receipt[]`,
    `branding` (or import `activeBranding` directly, like ReceiptPreview).
  - Renders the full estado de cuenta as one HTML "paper" (`.statement-paper`).
  - Section order + labels **must match** `generate_estado_de_cuenta_pdf` for parity:
    1. **PAQUETE** — items where `concept === 'PACKAGE'`
    2. **EXTRAS** — conceptual items (`HOSPITAL/HOSPITALIZACION/QUIROFANO/ANESTESIA/RAYOS_X/`
       `ULTRASONIDO/ROOM/ADVANCE/MEDICAMENTOS/MATERIAL_QUIRURGICO/OTROS`)
    3. **HONORARIOS MÉDICOS** — extras `category === 'FEES'`
    4. **LABORATORIOS** — extras `category === 'LABORATORY'`
    5. **GASTOS EXTRAS** — `OTHER`-concept items + extras `category === 'OTHER'`
    6. running **Total** (muted) → **GRAN TOTAL**
    7. **ANTICIPOS** — one row per receipt (N° · concepto · fecha · monto) → **TOTAL PAGADO** (teal)
    8. **SALDO PENDIENTE** (red, largest)
  - Exclude mirrored items: skip statement items whose `id` matches any `extra.statementItemId`
    (same rule the backend applies) so lab/fee/inventory charges don't print twice.
  - Each charge section = titled table: `Concepto/Bodega | Descripción | (qty × Q precio) | Monto`
    + a right-aligned **Subtotal** row.
  - Money: `Q {{ n.toLocaleString('es-GT', { minimumFractionDigits: 2 }) }}`, `font-mono`,
    `whitespace-nowrap`. Dates: reuse ReceiptPreview's `formatDate` (`es-GT`, long month).
  - `@media print` block: copy ReceiptPreview's visibility trick but scoped to `.statement-paper`;
    `@page { margin: 0 }`; paper `padding: 12mm`. Carta portrait (no fixed page size needed).

### Modify
- `modules/admissions/pages/AdmissionDetailPage.vue`
  - Render `<AdmissionStatementPreview>` once, hidden on screen / visible on print (same technique as
    receipts — the `@media print` visibility toggle already hides everything else).
  - **Keep BOTH buttons during comparison** (do not remove `downloadPdf`):
    - New: **"Imprimir (HTML)"** → `window.print()` (icon `Printer`).
    - Existing: keep **"Descargar PDF"** → `downloadPdf()` (server, icon `FileDown`) so the two
      outputs can be generated from the same admission and compared. `pdfLoading` stays.
  - Guard the print button disabled while `detail`/`extras`/`receipts` still loading, so print never
    fires on partial data.

## Edge cases
- Empty sections: render nothing for a section with 0 rows (matches `ec_section` early-return).
- No logo (`logoUrl == null`): show name+location only, no broken `<img>` (ReceiptPreview `v-if`).
- Overpayment: `pendingAmount <= 0` → SALDO in green/teal, not red (page already has this logic).
- Long tables spanning >1 print page: let the browser paginate; don't force `page-break`. Avoid
  `position: fixed` on the paper if content can exceed one page — ReceiptPreview uses `fixed` because
  a receipt is single-page; a statement may be multi-page, so use the visibility toggle WITHOUT
  `position: fixed` (keep paper in normal flow) so long statements paginate correctly.
- Print only the statement: `print:hidden` on all page chrome / the visibility trick already covers it.

## Existing patterns to follow
- `modules/receipts/components/ReceiptPreview.vue` — print CSS, header two-column, table styling,
  `formatDate`/`formatAmount`, `activeBranding` usage, `@inksightdev/ui` `Table*` components.
- Grouping/section semantics: `api/src/infrastructure/pdf/mod.rs::generate_estado_de_cuenta_pdf`
  (source of truth for order, labels, and the mirrored-item exclusion).

## SKILL_RECOMENDADA
None. In-repo `ReceiptPreview.vue` already covers the pattern; no installable skill needed.

## Diseño visual
Derive the statement's identity from `ReceiptPreview.vue` so receipt + statement read as **one
document family** (cohesion beats novelty for an internal financial document). Elevate it for a
full-page multi-section statement.

**Tokens (reuse existing palette):**
- Ink `#111827` (gray-900) · Muted `#6B7280` (gray-500) · Subtle `#9CA3AF` (gray-400)
- Accent `#0F766E` (teal-700) — section titles, "ESTADO DE CUENTA" badge, TOTAL PAGADO
- Negative `#DC2626` (red-600) — admission N° and SALDO PENDIENTE
- Strong rule gray-800; row separators gray-100/200; zebra `bg-gray-50` on alt rows
- Type: existing UI sans; **`font-mono` for every amount**, right-aligned, `tabular-nums`

**Header (per request):** left = tenant logo **large, ~h-20/h-24** (roughly the height of the
name+location block, noticeably bigger than the receipt's `h-14`) + `object-contain`, beside it
clinic **name** (uppercase, `tracking-widest`, gray-900) and location line (`CLÍNICA TEST` /
`Guatemala, Guatemala`, gray-400). Right = teal outlined badge "ESTADO DE CUENTA", `N° <admission>`
in red, statement date. Strong `border-b-2 border-gray-800` under the header.

**Signature element:** the **SALDO PENDIENTE** block — the one bold moment. Right-aligned, largest
type on the page, red when owed / teal when settled, sitting above a hairline footer. Everything
else stays quiet and disciplined (thin rules, generous whitespace, muted section labels).

**Structure encodes meaning:** section titles (PAQUETE / HONORARIOS / LABORATORIOS…) are the real
grouping, not decoration; each closes with its Subtotal so the eye can audit the running total →
GRAN TOTAL → TOTAL PAGADO → SALDO chain top to bottom. No numbered markers (not a sequence).

**Meta grid** under the header (2 cols): PROCEDIMIENTO / HABITACIÓN / FECHA DE INGRESO |
MÉDICO-CIRUJANO / AYUDANTE / ANESTESIÓLOGO — tiny uppercase muted labels over gray-900 values.

**Print discipline:** black-on-white, no shadows/rounded on print (`print:shadow-none
print:border-none`), respect page margins via `@page`. No animation — static document; `/apple-design`
(gestures/springs) is **N/A** here.

# Approach B — server `genpdf` restyle (backend)

Restyle the existing PDF **in place** — same endpoint, same data, same `genpdf` pipeline. Goal: make
the current output read like a statement, not a raw dump, within genpdf's real limits.

## Honest ceiling (state up front)
`genpdf` 0.2 has **no cell background fills** → no zebra striping, no filled header/total bands, no
rounded corners. Borders are limited to `FrameCellDecorator` rules (horizontal/vertical/outer). So
Approach B can improve hierarchy, spacing, logo, and framing — but cannot match Approach A's fills.
This asymmetry is the whole point of the comparison.

## Files — Modify only
- `hospital-belen-api/src/infrastructure/pdf/mod.rs` → `generate_estado_de_cuenta_pdf` (+ shared
  helpers `make_pdf_logo`, `ec_section*`, header block). No new deps, no signature changes.

## Concrete changes (do all; each is within genpdf)
1. **Header logo — large, covers title height.** Today `make_pdf_logo` caps at 16 mm tall and the
   logo is stacked *above* the text block. Change to: logo **beside** the identity text in a 2-cell
   row (`TableLayout::new(vec![logo_w, text_w])`), target height **~24–26 mm** (≈ height of
   name+address block), width clamp raised proportionally. Add a `logo_h_mm` param to `make_pdf_logo`
   (default keeps receipt at 16 mm; statement passes the larger value) so the receipt PDF is unaffected.
2. **Location line in header.** Already have `tenant_address`; render clinic **name** (bold 15) then
   **address/location** (`CLÍNICA TEST` context → tenant name + `Guatemala, Guatemala`) directly under
   it, matching the request's header content.
3. **Section framing.** Give each charge section (`ec_section`, `ec_section_conceptual`) a stronger
   title treatment (teal bold, a full-width top+bottom rule via `FrameCellDecorator::new(false,true,false)`
   already present) and consistent right-aligned money column. Add a thin rule under each section's
   Subtotal so sections read as discrete blocks.
4. **Totals hierarchy.** Keep GRAN TOTAL / TOTAL PAGADO (teal) / SALDO PENDIENTE (red, largest) but
   frame the SALDO row with an outer border box (`FrameCellDecorator::new(true,true,true)` on a
   single-row table) to make it the visual anchor — the genpdf equivalent of Approach A's signature block.
5. **Spacing pass.** Tighten `Break` values into a consistent rhythm; align all `Q x.xx` columns to
   the same right edge across every section.

## Edge cases (B)
- Empty sections already early-return (`ec_section` guards `rows.is_empty()`) — keep.
- WebP logo without the `webp` image feature still returns `None` (existing `make_pdf_logo` behavior)
  — larger target size must not change that fallback.
- Regenerate nothing DB-side; this is pure rendering. No sqlx changes → no `.sqlx` cache impact.

# Comparison & Decision

After both are built, the tester/human generates **the same admission** through both paths and
compares:
- HTML: "Imprimir (HTML)" → browser Print → Save as PDF.
- Server: "Descargar PDF".
Reference admission: `162a3a09-1f3a-4f39-9abc-83a47c408727` (the URL in the request) — has real
sections to exercise. Compare on: header/logo presence, table legibility, section separation, totals
emphasis, multi-page behavior, fidelity to a "real invoice/statement".
**Decision is the human's** — record it in `review.md`. The losing approach (and its dead code:
either the HTML component + print button, or the genpdf restyle + server button/endpoint) is removed
in a **follow-up fix task**, not here. // ponytail: don't pre-delete either path before the human picks.

# DONE checklist for coder
Approach A (HTML):
- [ ] `AdmissionStatementPreview.vue` renders all 8 sections with correct grouping + mirrored-item exclusion.
- [ ] "Imprimir (HTML)" prints the statement only; multi-page paginates (no `position: fixed` on paper).
- [ ] Logo-absent, empty-section, and overpayment cases handled.
Approach B (genpdf):
- [ ] Header logo enlarged (~24–26 mm, beside text) + location line; receipt PDF unchanged.
- [ ] Section framing + SALDO boxed; totals hierarchy intact; amounts right-aligned consistently.
Both:
- [ ] BOTH buttons present and functional on `AdmissionDetailPage.vue`; same admission drives both.
- [ ] `cargo clippy --all-targets -- -D warnings` clean (backend gate); web typechecks/builds.
