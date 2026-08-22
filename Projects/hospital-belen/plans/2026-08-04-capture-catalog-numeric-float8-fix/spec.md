# Spec — capture_snapshot 500: NUMERIC→f64 decode in capture_catalog (APP bug)

Branch: `fenix/modules-i18n-admissions-cleanup` @ `e07be2e` (PR #19).

## Root cause (confirmed) — this is an APP bug, not a test bug
`POST /api/platform/tenants/{id}/snapshots` returns 500 because
`tenant_seed::capture_catalog` (`src/infrastructure/tenant_seed.rs:791`) SELECTs Postgres
`numeric` columns and decodes them straight into Rust `f64`, which sqlx **cannot decode**
without the `rust_decimal`/`bigdecimal` feature — and those features are NOT enabled
(`Cargo.toml:39` sqlx features = runtime-tokio, tls, postgres, uuid, time, macros, json). sqlx
throws `mismatched types; Rust type f64 is not compatible with SQL type NUMERIC` → `DomainError`
→ 500.

Columns affected (all `numeric`, all NOT NULL — so it's a type-decode failure, not a NULL issue):
- `products.default_price` → decoded as `f64` (tenant_seed.rs ~805)
- `medical_packages.base_cost` → `f64` (~862)
- `package_items.quantity`, `package_items.unit_price` → `f64` (~876)

**Why it only fails for some tests (the tell):**
- PASS `capture_catalog_creates_snapshot` / `recapture_same_name` — capture `"inventory"` from a
  freshly `provision_tenant()`ed **empty** tenant → 0 product rows → the decode never runs → 201.
- FAIL `apply_snapshot_to_tenant_creates_catalog`, `apply_private_snapshot…403`,
  `remove_seed_with_in_use_product…409` — capture `"packages"` from **hospital-belen**, which has
  seeded products/packages → real rows → decode fires → 500.
The tests are correct (the round-16 P4 fix made asserts unconditional, which is exactly why this
real 500 now surfaces instead of being silently skipped). "capture body validation missing" is a
red herring — module/visibility validation IS present (platform_seed.rs:206-217); the 500 is the
NUMERIC decode.

## The established pattern in this codebase
Every other query that reads a `numeric` into `f64` casts it `::float8`:
`inventory_repository.rs:179,247-250`, `report.rs:32,70`, `dashboard.rs:56`,
`admission_repository.rs:80-81`, `admission_extra_repository.rs:43-44`, etc. `capture_catalog` is
the one place that omitted the cast.

## Fix (src/infrastructure/tenant_seed.rs::capture_catalog — add `::float8` casts)
- Products SELECT: `p.default_price` → `p.default_price::float8`
- Packages SELECT: `base_cost` → `base_cost::float8`
- Package-items SELECT: `pi.quantity` → `pi.quantity::float8`, `pi.unit_price` →
  `pi.unit_price::float8` (leave `pi.sort_order` — it's `integer`→`i32`, no cast)
Tuple decode types (`f64`) stay as-is; only the SQL gains the casts. No dependency/feature change,
no schema change — this matches the rest of the codebase.

## Guard against regression
`capture_catalog` was only ever exercised against empty tenants in CI, so the bug hid. Ensure at
least one test captures a NON-empty tenant per module: keep `apply_snapshot_to_tenant_creates_catalog`
capturing `"packages"` from hospital-belen, and add/keep an `"inventory"` capture from hospital-belen
too (it has inventory seed data) so the products-cast path is covered with real rows. Consider a
`"full"` capture from hospital-belen as the belt-and-suspenders case.

## Verify
```bash
cargo clippy --all-targets -- -D warnings && cargo fmt --all -- --check
cargo test --all-targets --no-fail-fast     # GREEN and STABLE across 3 back-to-back runs
```
The three failing tests must go 201 on capture and then pass their apply/remove assertions.

## Files
- `src/infrastructure/tenant_seed.rs` (capture_catalog — 3 SELECTs get `::float8`) — **product code**
- (optional) `tests/seed_snapshots.rs` — add a non-empty inventory/full capture for coverage

## Notes
- This round DOES touch `src/` — it's a genuine handler-path bug, not a test defect.
- No rebase/cherry-pick. No new deps. branch.md unchanged. Run all three check gates before push.
