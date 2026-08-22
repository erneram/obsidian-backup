VERDICT: SHIP (code quality) — manual browser gate required for final acceptance

## Razonamiento
- Full round on new branch fenix/tenant-name-receipts-package-modal. (.pipeline/fix-request.md is
  STALE from the old modules-i18n branch — remove_seed, already fixed — ignored.)
- T1/T2/T3 all match spec; no dead wiring, no over-deletion, backend fmt+clippy clean.

## T1 — tenant name (correct)
- API (substantive): download_statement_pdf (admission.rs) + download_receipt_pdf (receipt.rs) both
  SELECT name,address FROM tenants WHERE id=$1 scoped to ctx.tenant_id(); NULL address → fallback;
  missing row → old literals. .sqlx cache committed (one entry, both handlers share identical query).
- Web: ReceiptPreview header activeBranding?.name ?? 'Hospital Belén'; footer parametrized via
  $t(...,{name}). activeBranding is a module-level ref, but ReceiptPreview is <script setup> → the
  compiler unrefs imported bindings in the template (_unref), so ?.name resolves correctly. Not a bug.

## T2 — receipt cleanup (correct)
- "Descargar PDF" button + dead wiring (downloadLoading prop, downloadPdf emit, Download/Loader2
  imports, ReceiptDetailPage @download-pdf handler) removed. Correctly PRESERVED list-flow usage
  (ReceiptList download button + useReceipt().downloadPdf still used by ReceiptListPage) — no
  over-deletion. Signature spacing py-6→pt-12 pb-6, w-40→w-56, mt-10 above each rule.

## T3 — package modals (correct)
- Create (PackageListPage) + Edit + Add-Item (PackageDetailPage): max-w-md→max-w-2xl mx-4
  max-h-[85vh] overflow-y-auto. 3 panels, matches spec.

## Hallazgos
🟡 MEDIO — OPEN QUESTION unresolved: T3 targeted the create/edit/add-item modals. The spec flagged
   that "package list scrollable" might instead mean the "Aplicar Paquete Médico" picker in
   AdmissionDetailPage (~496), which is untouched. Human should confirm which was intended; if the
   picker, it's a small follow-up (same cap-height+scroll).
🟢 BAJO — backend style inconsistency: admission uses fetch_optional().ok().flatten(); receipt uses
   fetch_one()+if let Ok. Both correct/graceful; harmless.

## Manual gate (REQUIRED before /approve — I cannot run browser/dev-server)
- Render receipt & Estado de Cuenta PDFs for a NON-Belén tenant → header/footer show that tenant's
  real name+address.
- ReceiptPreview: no "Descargar PDF" button; visible signing space above each signature line; print
  keeps two columns.
- Package create/edit/add-item modals: scroll INSIDE the panel at ~375px and desktop; fields don't overflow.

## PRs
API: https://github.com/InkSight-Developments/hospital-belen-api/pull/20
WEB: https://github.com/InkSight-Developments/hospital-belen-web/pull/29
MAIN: https://github.com/InkSight-Developments/hospital-belen/pull/20
