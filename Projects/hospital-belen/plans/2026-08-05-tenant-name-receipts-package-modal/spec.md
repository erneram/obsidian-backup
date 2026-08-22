# Spec — Dynamic tenant name + receipt/package UI polish

Branch (both repos): `fenix/tenant-name-receipts-package-modal`. New work.
Frontend = `hospital-belen-web/kairosaid`. Backend = `hospital-belen-api`.

---

## Task 1 — Replace hardcoded "Hospital Belén" with the actual tenant name

### Audit result (whole codebase, both repos)
| Location | Kind | Action |
|----------|------|--------|
| `web …/receipts/components/ReceiptPreview.vue:34` (`<h1>Hospital Belén</h1>`) | on-screen + printed receipt header | **FIX** → tenant name |
| `web …/locales/es.json:216` `receipts.preview.footer` = "Hospital Belén — Documento oficial" | printed receipt footer | **FIX** → parametrize |
| `api src/infrastructure/pdf/mod.rs:93` `tenant_name: "Hospital Belén"` (+`:94` address) | **server-generated receipt PDF** | **FIX** → tenant lookup |
| `api src/web/handlers/admission.rs:388` `tenant_name: "Hospital Belén"` (+`:389` address) | **server-generated Estado de Cuenta PDF** | **FIX** → tenant lookup |
| `web …/platform/components/TenantChannelsForm.vue:24`; `admin/pages/TenantListPage.vue:60,71` | input **placeholders** in admin forms | leave (not display) |
| `api tenant_seed.rs:120`, `dev_seed.rs:*` | seed data / dev fixtures | leave (that IS the Belén tenant) |

Key finding: the **Estados de Cuenta and receipt PDFs are generated server-side and hardcode the
name for EVERY tenant** — that's the most important part of this task, and it's a backend fix.

### Frontend fix (web)
Tenant name is available app-wide via `activeBranding` (`composables/useBranding.ts`, a
`TenantBranding` with `{ slug, name, logoUrl }`, populated in `main.ts` on startup).
- `ReceiptPreview.vue:34`: `<h1 …>Hospital Belén</h1>` →
  `<h1 …>{{ activeBranding?.name ?? 'Hospital Belén' }}</h1>` and
  `import { activeBranding } from '@/composables/useBranding'`.
- Footer: change `es.json` `receipts.preview.footer` to `"{name} — Documento oficial"` and render
  `$t('receipts.preview.footer', { name: activeBranding?.name ?? 'Hospital Belén' })`
  (ReceiptPreview.vue:113).
- Line 36 `Guatemala, Guatemala` and line 35 subtitle are generic — leave unless the human wants
  address dynamic on the on-screen receipt too (OPEN QUESTION below).

### Backend fix (api) — the official PDFs
Both PDF builders hardcode `tenant_name`/`tenant_address`. Replace with the real tenant row:
- `SELECT name, address FROM tenants WHERE id = $1` bound to `ctx.tenant_id()` (the caller already
  has `ctx`/`State` in both handlers; `admission.rs` builds `pdf_data` in the handler,
  `pdf/mod.rs` builds it in a `From<Receipt>` impl — thread the tenant name/address in from the
  handler rather than inside the `From` impl so it has DB access).
- `pdf/mod.rs:93-94` and `admission.rs:388-389`: use the looked-up `name`/`address`, fallback to
  the current literals only if the row is missing.
- `tenants.address` may be NULL — `COALESCE(address, 'Guatemala, Guatemala')`.

---

## Task 2 — Receipt view (`ReceiptPreview.vue`)
1. **Remove "Descargar PDF" button.** Delete the second `<Button>` (lines 13-22, the
   `@click="$emit('downloadPdf', …)"` one). Keep "Imprimir" (lines 5-12). Then clean the now-dead
   code: `downloadLoading` prop, `downloadPdf` emit, `Download`/`Loader2` imports, and in the
   parent `ReceiptDetailPage.vue` the `@downloadPdf` handler + `useReceipt().downloadPdf` if unused
   (ponytail — don't leave dead wiring). The server receipt-PDF endpoint can stay; it's just no
   longer surfaced.
2. **Signature spacing** (lines 100-115, `authorizedSignature` / `patientSignature`). Give the
   signatures room to actually be signed: increase the block's top padding and the space above each
   signature line. Concretely: container `py-6` → `pt-12 pb-6`; add height above the rule so there's
   a blank signing area (e.g. wrap each in a taller box or add `mt-10` above the `border-b` line);
   optionally widen `w-40` → `w-56`. Keep the two columns side-by-side on print; stack acceptably on
   narrow screens.

---

## Task 3 — Package creation modal (`packages/pages/PackageListPage.vue:205-247`)
Current create modal: `<div class="bg-card rounded-lg shadow-xl w-full max-w-md p-6 space-y-4">` —
too narrow (`max-w-md`) and no internal scroll, so tall content / small viewports push the whole
page instead of scrolling inside.
- **Widen:** `max-w-md` → `max-w-2xl` (responsive: keep `w-full` and add `mx-4` so it fits mobile).
- **Scroll inside, not the page:** cap height and scroll the body —
  `max-h-[85vh] overflow-y-auto` on the panel (or keep header/footer fixed and put
  `overflow-y-auto` on a middle body div). The backdrop already `fixed inset-0 flex items-center
  justify-center`, so a capped panel stays centered.
- **Responsive check:** verify at ~375px and desktop; the Select/Textarea fields must not overflow.
- Apply the same `max-h-[85vh] overflow-y-auto` + width bump to the sibling package dialogs for
  consistency: `PackageDetailPage.vue:305` (Edit) and `:348` (Add Item).

**OPEN QUESTION:** the task says "make the *package list* scrollable." The literal creation modal
(above) is a short form with no long list. If you instead mean the **"Aplicar Paquete Médico"**
picker in `AdmissionDetailPage.vue` (~line 496, which lists selectable packages), say so — that's
the only package-*list* modal, and the same cap-height + scroll fix applies there.

---

## Diseño visual
- Receipt signatures: aim for a real signing gap — ~40-56px of clear space above each ruled line,
  labels muted (`text-gray-400`) below. Preserve the print layout (two columns).
- Package modal: centered, `max-w-2xl`, panel scrolls internally at `max-h-[85vh]`; never let the
  backdrop-covered page scroll behind it. Match existing modal styling (`bg-card rounded-lg
  shadow-xl`). frontend-design skill not deeply invoked — these are polish tweaks, not new UI.

## Files
Frontend: `receipts/components/ReceiptPreview.vue`, `receipts/pages/ReceiptDetailPage.vue`,
`locales/es.json`, `packages/pages/PackageListPage.vue`, `packages/pages/PackageDetailPage.vue`
(consistency). Backend: `src/infrastructure/pdf/mod.rs`, `src/web/handlers/admission.rs`
(+ tenant lookup helper if none exists).

## Notes
- Task 1 spans BOTH repos (frontend header/footer + backend official PDFs) — the PDF fix is the
  substantive one.
- Backend touches SQL/handlers → run `cargo clippy --all-targets -- -D warnings` + `cargo fmt`
  before push (memory `hospital-belen-ci-clippy-gate`). Reuse tenant name via `activeBranding` on
  the web side — no new API call needed there.
- SKILL_RECOMENDADA: none.
