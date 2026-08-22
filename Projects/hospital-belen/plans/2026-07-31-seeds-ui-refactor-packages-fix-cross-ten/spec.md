# Spec — Seeds page UI + cross-tenant package seeding review

Three independent deliverables:
- **A.** Migrate the seeds page raw HTML controls to `@inksightdev/ui` components.
- **B.** Fix a real correctness gap: capturing a `packages` snapshot drops storages, so
  applying it to another tenant silently loses every package item. **(Approved — proceed.)**
- **C.** Convert the hardcoded hospital-belen reference catalog in
  `migrations/003_seed_reference.sql` (lines 42–3577) into a reusable builtin seed that applies
  to any tenant through the existing seed framework.

---

## A. Seeds page — use `@inksightdev/ui` controls

**File:** `hospital-belen-web/kairosaid/src/modules/platform/pages/PlatformSeedsPage.vue`

Today it uses raw `<select>` / `<input>` for the tenant selector and the capture modal.
The page already imports `Dialog`, `Button`, `Checkbox`, `toast` from `@inksightdev/ui` — extend
the import with `Select, SelectTrigger, SelectValue, SelectContent, SelectItem, Input, Label`.

**Pattern to follow (identical use case, tenant selector):**
`src/modules/platform/pages/PlatformRolesPage.vue:17-23`
```html
<Select v-model="selectedTenantId" @update:model-value="onTenantChange">
  <SelectTrigger class="w-full sm:w-52">
    <SelectValue placeholder="…" />
  </SelectTrigger>
  <SelectContent>
    <SelectItem v-for="t in tenants" :key="t.id" :value="t.id">{{ t.name }}</SelectItem>
  </SelectContent>
</Select>
```

### Changes

1. **Tenant selector** (lines 21-28): replace the raw `<select v-model="selectedTenantId">`
   with the `Select` block above. Keep `@update:model-value="onTenantChange"` (drop the old
   `@change`). Keep the `<label>`/`Label` above it with `$t('platform.seeds.tenant')`.
   Placeholder = `$t('platform.seeds.selectTenant')`. Do **not** keep the empty-value `<option>` —
   `Select` uses the placeholder for the empty state.

2. **Capture modal — name field ("box field")** (lines 159-164): replace raw `<input v-model="captureForm.name">`
   with `<Input v-model="captureForm.name" required placeholder="e.g. Catalog Q1 2026" />`.
   Wrap its label in `<Label>`.

3. **Capture modal — module select** (lines 168-175): replace with `Select` bound to
   `captureForm.module`, options `inventory | packages | full`.

4. **Capture modal — visibility select** (lines 179-185): replace with `Select` bound to
   `captureForm.visibility`, options `PRIVATE | SHARED`.

### Edge cases
- `onTenantChange` currently keys off `@change`; `Select` emits `update:model-value` — verify the
  handler still fires on selection and on clearing.
- `selectedTenantId` empty string drives `v-if="selectedTenantId"` blocks — keep the empty-string
  default so the catalog stays hidden until a tenant is picked.
- Match dark-mode classes already used by `Select` in `PlatformRolesPage.vue`; don't hand-roll
  `border-input`/`bg-background` styling.

Existing pattern: `PlatformRolesPage.vue` (tenant `Select`), any form page using `Input`
(e.g. `src/components/patients/PatientFormStep1.vue:12`).

---

## B. Cross-tenant package seeding — correctness fix

**The user's report is valid.** The mechanism to seed another tenant with hospital-belen's
packages exists (capture snapshot → SHARED → share/apply), but it is broken for the
`packages` module.

**Root cause** — `hospital-belen-api/src/infrastructure/tenant_seed.rs`:

- `capture_catalog` (line ~783): for `module = "packages"` it captures `products`, `packages`,
  and `package_items`, but **not** `warehouses` / `storages` (those are gated behind
  `include_inventory`, lines 813-842).
- `apply_catalog` inserts `package_items` by joining on an existing storage in the **target**
  tenant: `... FROM medical_packages mp, products p, storages s WHERE ... s.code = $4` (lines 657-682).
  If the target tenant has no storage with that `code`, the row matches nothing and the item is
  **silently skipped** — no error. The package header is created with zero items.
- Newly created tenants are **not** baseline-seeded (`seed_tenant_catalog` runs only in
  `dev_seed.rs:64-65`; the platform tenant-create path in `web/handlers/tenant.rs` does not call it),
  so a fresh target tenant has no storages at all → every package item is dropped.

**Fix:** when capturing `packages`, also capture the warehouses/storages referenced by the
captured `package_items`, so applying the snapshot recreates them in the target tenant.

In `capture_catalog`, compute inclusion as:
```rust
let include_inventory = matches!(module, "inventory" | "full");
let include_packages  = matches!(module, "packages" | "full");
// packages need their storages (+ parent warehouses) to survive a cross-tenant apply
let need_storages = include_inventory || include_packages;
```
Then gate the warehouse + storage capture blocks (lines 813-842) on `need_storages` instead of
`include_inventory`. `apply_catalog` already inserts warehouses/storages before package_items and
is idempotent (`ON CONFLICT DO NOTHING`), so no apply-side change is needed.

Scope note: a `packages` snapshot will now also carry the warehouses/storages its items touch. That
is correct — the items are unusable without them. Products are already captured for `packages`.

### Edge cases to cover
- **Fresh target tenant** (no warehouses/storages): after the fix, applying a `packages` snapshot
  must create the referenced warehouses, storages, products, package, and all package_items.
- **Partial-storage target** (some storage codes exist, some don't): all referenced storages get
  created; no duplicate on the existing ones (`ON CONFLICT (warehouse_id, code) DO NOTHING`).
- **`packages` snapshot referencing a storage whose warehouse isn't otherwise captured**: ensure the
  parent warehouse of every referenced storage is included, or the storage insert (which joins on
  `warehouses w ... w.code = $2`) is itself silently skipped — same class of bug one level up.
  Capture all warehouses+storages of the tenant (simplest, matches current inventory capture) rather
  than trying to filter to only referenced ones.
- **Idempotent re-apply / share to same tenant twice**: still no duplicates.
- **`applied_seeds` marker** (line 464): unaffected — snapshot presence is keyed off first package
  code, which still exists.

### Suggested verification (for tester)
- Capture a `packages` snapshot from a tenant that has a package with items; assert the payload JSON
  now contains non-empty `warehouses` and `storages`.
- Apply it to a fresh tenant (no baseline seed); assert `package_items` count for that tenant > 0.

---

---

## C. Reusable hospital-belen catalog seeder

**Goal:** the full hospital-belen surgery catalog (112 products, 116 packages, 2726 package_items,
4 warehouses, 4 storages) must be applyable to **any** tenant from the seeds page — the same way
the small builtin `packages.pkg_cir_001` / `packages.pkg_par_001` seeds already are.

**Why the migration can't be reused as-is:**
`hospital-belen-api/migrations/003_seed_reference.sql:42-3577` hardcodes
`tenant_id = '10000000-0000-0000-0000-000000000001'` and fixed UUID5 ids on every row. It only ever
seeds the Belén tenant. (Lines after 3577 are WHO growth tables — **out of scope**, leave them.)

**Approach — register it as a builtin seed, data loaded from a bundled JSON asset.**
The existing `apply_catalog` (`tenant_seed.rs`) is already tenant-parameterized, keyed by `code`
(not id), and idempotent — it lets the DB assign per-tenant ids. Reuse it verbatim; only supply it a
`SeedCatalog` built from the Belén data. Hardcoding 2726 items in Rust is unacceptable, so the data
lives as a JSON asset matching `SeedCatalog`'s serde shape.

### Changes

1. **Generate the JSON asset.** `hospital-belen-api/assets/seeds/belen_surgery.json` — a `SeedCatalog`
   (`products`, `warehouses`, `storages`, `packages`, `package_items`) with **no** `tenant_id`/`id`
   fields (codes only), exactly the shape `SeedCatalog` already deserializes.
   - Primary path: extend `scripts/gen_surgery_seed.py` (the generator that produced the SQL, source
     = Excel in `modelosDeCirugias/`) to also emit this JSON. Single source of truth.
   - Fallback path (if the Excel/py env isn't runnable): a one-off parser that reads the existing
     `003_seed_reference.sql:42-3577` INSERTs and emits the JSON. Verify counts: 112 products /
     116 packages / 2726 package_items / 4 warehouses / 4 storages.

2. **Register the builtin seed** in `tenant_seed.rs` `SEEDS`:
   ```rust
   BuiltinSeed {
       id: "packages.belen_surgery",
       name: "Catálogo Hospital Belén — cirugías",
       module: "full",                    // ships products + storages + packages
       marker_package: Some("<a representative package code from the catalog>"),
       marker_product: None,
       catalog: build_belen_surgery,
   }
   ```
   ```rust
   fn build_belen_surgery() -> SeedCatalog {
       // ponytail: parse the bundled asset each call; ~1MB JSON, seeds run rarely.
       // upgrade to OnceLock cache only if apply latency ever matters.
       serde_json::from_str(include_str!("../../assets/seeds/belen_surgery.json"))
           .expect("belen_surgery.json is a valid SeedCatalog")
   }
   ```
   `catalog: fn() -> SeedCatalog` — this signature already fits; no framework change.

3. **No handler/UI change needed.** It auto-appears in `list_seed_catalog` under module `full`,
   is applyable via the existing `apply_seeds` path, and (once deliverable A lands) is selectable
   in the seeds page. `applied_seeds` marker logic already handles builtin seeds by
   `marker_package`.

### Edge cases to cover
- **`unit_abbr` coverage:** every product's unit must exist in `units`. `apply_catalog`'s product
  INSERT joins `units u WHERE u.abbreviation = $6` and silently skips on miss. `ensure_service_units`
  seeds `hora`/`noche`; confirm the catalog uses only `unid`/`hora`/`noche` (or extend
  `ensure_service_units`). List any other abbreviations found during generation.
- **Storage/warehouse completeness:** the JSON must include every warehouse+storage referenced by
  `package_items.storage_code` (SALA-OP/OR, ENF/ENF-GEN, PISOS/PISOS-GEN, SERV/SERVICIOS) — same
  silent-skip failure mode as deliverable B if missing.
- **Idempotency:** applying twice, or to a tenant that already has some of these codes, must not
  duplicate (all inserts are `ON CONFLICT … DO NOTHING` / `NOT EXISTS`).
- **Duplicate/dirty names in source data** (e.g. `INS-0009` "Ampollas de Dolofor  o Doclofenaco" vs
  `INS-0010`): keep codes exactly as in the SQL — codes are the dedup key, don't normalize.
- **Marker choice:** pick a package code guaranteed present in the catalog so `applied_seeds`
  reports it correctly; document which one.
- **Asset must ship in the build:** `include_str!` embeds at compile time, so no Docker/runtime asset
  wiring needed — but confirm the path is correct relative to `src/infrastructure/tenant_seed.rs`.

### Suggested verification (for tester)
- Deserialize `belen_surgery.json` into `SeedCatalog` in a unit test; assert element counts
  (112 / 116 / 2726 / 4 / 4).
- Apply `builtin.packages.belen_surgery` to a fresh tenant; assert products=112, medical_packages=116,
  package_items=2726 for that tenant, and that `applied_seeds` returns the seed id.

---

Interaction between B and C: C ships as module `full` (includes storages), so it is unaffected by
the B bug. B still needed for user-captured `packages` snapshots.

SKILL_RECOMENDADA: none applicable.
