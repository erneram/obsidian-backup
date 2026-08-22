# Changes — fenix/tenant-name-receipts-package-modal

## Files changed

### API (`hospital-belen-api`)
- `src/web/handlers/receipt.rs`: query tenant name/address from DB before building `ReceiptPdfData`; overrides hardcoded "Hospital Belén"
- `src/web/handlers/admission.rs`: single `SELECT name, address FROM tenants WHERE id = $1` before building `EstadoDeCuentaPdfData`; overrides hardcoded literals with fallback
- `.sqlx/`: updated offline query cache (`cargo sqlx prepare`) for new queries

### Web (`hospital-belen-web/kairosaid`)
- `src/modules/receipts/components/ReceiptPreview.vue`:
  - Header h1 now reads `activeBranding?.name ?? 'Hospital Belén'`
  - Footer uses `$t('receipts.preview.footer', { name })` for parametrized string
  - Removed "Descargar PDF" button, `downloadLoading` prop, `downloadPdf` emit, `Download`/`Loader2` imports
  - Signature section: `py-6` → `pt-12 pb-6`; `w-40` → `w-56`; `mt-10` added above each `border-b` rule
- `src/modules/receipts/pages/ReceiptDetailPage.vue`:
  - Removed `@download-pdf` handler, `handleDownloadPdf` fn, `useReceipt` import, pdfError div
- `src/locales/es.json`: `receipts.preview.footer` → `"{name} — Documento oficial"`
- `src/modules/packages/pages/PackageListPage.vue`: Create modal `max-w-md` → `max-w-2xl mx-4 max-h-[85vh] overflow-y-auto`
- `src/modules/packages/pages/PackageDetailPage.vue`: Edit + Add Item modals same width/scroll fix

## Repos tocados
- `api` (hospital-belen-api)
- `web` (hospital-belen-web)

## What the Tester should review
- Receipt PDF download: tenant name in PDF header should match the tenant's actual name (not "Hospital Belén")
- Estado de Cuenta PDF: same
- Receipt preview on screen: h1 shows tenant name; footer shows `{tenant} — Documento oficial`
- No "Descargar PDF" button visible in ReceiptPreview
- Signature block has visible signing space above each line
- Package create/edit/add-item modals: wider, scroll internally at tall viewports
