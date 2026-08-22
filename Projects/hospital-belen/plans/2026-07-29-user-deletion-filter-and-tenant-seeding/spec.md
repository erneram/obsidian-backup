# Spec — hospital-belen (round 3)

OPEN QUESTION (Part 2): Is the tenant seeder **dev-only** (like `dev_seed.rs`, runs on boot when
`APP_ENV != production`) or **production tenant-onboarding** (triggered when a new tenant is
created, seeding a baseline catalog)? And what is the data source for inventories + paquetes
médicos — a fixed built-in baseline catalog, or copied from an existing template tenant? Spec
below assumes **built-in baseline catalog, callable from both dev seed and a future onboarding
hook**. Confirm before Coder implements Part 2.

---

# Part 1 — GET /users list still returns soft-deleted (INACTIVE) users

## Problem
`DELETE .../users/{id}` now returns 204 and sets `status = 'INACTIVE'` (soft delete works), but the
deleted user still appears in `GET /users`. The list query does not exclude soft-deleted rows.

## Root cause
`base::list` (`src/infrastructure/database/base.rs`) only excludes soft-deleted rows for the
`is_deleted` flag pattern:
```rust
if excludes_soft_deleted_flag(MC::DELETE_MODE) {   // matches ONLY DeleteMode::SoftIsDeleted
    query.and_where(Expr::col(SoftDeleteIden::IsDeleted).eq(false));
}
```
`UserBmc` uses `DeleteMode::SoftSetField { column: "status", value: "INACTIVE", cast: Some("user_status") }`
(the enum-cast `cast` field was added in round-1's delete fix). That mode is **not** handled here,
so INACTIVE users leak into every `list`/`count`. `base::count` has the same gap.

## Fix — `src/infrastructure/database/base.rs`
Add a `SoftSetField` exclusion in **both** `list` and `count`, mirroring the existing `is_deleted`
exclusion but with the enum cast:
```rust
if let DeleteMode::SoftSetField { column, value, cast } = MC::DELETE_MODE {
    let excluded = match cast {
        Some(ty) => Expr::val(value).cast_as(Alias::new(ty)),
        None => value.into(),
    };
    query.and_where(Expr::col(Alias::new(column)).ne(excluded));
}
```
Place it right after the existing `excludes_soft_deleted_flag(...)` block in `list`, and the
equivalent spot in `count`. Reuse the same cast helper shape used by `_soft_delete_set_field`.

## Edge cases
- `cast: None` → plain `column <> value` (no regression for non-enum future users).
- **Unconditional exclusion**: after this, the list endpoint can never show INACTIVE users, even
  with an explicit `status` filter. This matches soft-delete semantics ("deleted = gone") and the
  existing `is_deleted` behavior. NOTE: users set INACTIVE via the status toggle (update, not
  delete) also disappear from the list. If an "include inactive / show deactivated" admin view is
  wanted later, that needs an opt-in flag — **out of scope here**.
- `count` must apply the same predicate so pagination totals match the visible rows.

## Verification
- `cargo build`.
- Soft-delete a user → `GET /users` no longer lists them; `count` drops by 1.
- A still-ACTIVE user remains listed.

---

# Part 2 — Reusable tenant seeding module (inventories + paquetes médicos)

## Goal
A tenant-parameterized seeding module that populates a baseline catalog for **any** tenant (not
hardcoded to hospital-belen): products/inventory and medical packages (`paquetes médicos`).
Extensible = one entry point taking `tenant_id`, driven by a data catalog, reused per tenant.

## Existing patterns to follow
- `src/infrastructure/dev_seed.rs` — idempotent `INSERT ... ON CONFLICT (tenant_id, code) DO NOTHING`,
  per-tenant helper fns taking `tenant_id: Uuid, seeder_id: Uuid` (see `ensure_test_warehouses`).
- Relevant tables (`migrations/001_schema.sql`): `products` (466), `warehouses` (783),
  `inventory_items` (813), `medical_packages` (845, has `type package_type`, `created_by`),
  `package_items` (860, links package→product→storage). All keyed `UNIQUE (tenant_id, code)`.
- Prior art for package data: `migrations.bak/003_seed_surgery_packages.sql`.

## Proposed structure (pending OPEN QUESTION)
New module `src/infrastructure/tenant_seed.rs`:
```rust
/// Seed a tenant's baseline catalog. Idempotent — safe to re-run.
pub async fn seed_tenant_catalog(db: &Db, tenant_id: Uuid, seeder_id: Uuid) -> Result<()> {
    seed_products(db, tenant_id).await?;          // products + units refs
    seed_inventory(db, tenant_id, seeder_id).await?;  // warehouses/storages/inventory_items
    seed_medical_packages(db, tenant_id, seeder_id).await?; // medical_packages + package_items
    Ok(())
}
```
- Baseline data as `const`/static slices at the top of the module (code, name, category, price…),
  so adding a tenant = call `seed_tenant_catalog(db, tenant_id, seeder_id)`; adding an item = edit
  the slice. No per-tenant branching.
- Each `seed_*` uses `INSERT ... ON CONFLICT (tenant_id, code) DO NOTHING` then re-selects ids for
  FK wiring (package_items need product_id + storage_id), exactly like `ensure_test_warehouses`.
- `package_items.created` needs a valid `storage_id`; seed a default warehouse+storage first, then
  reference it — order matters.

## Wiring
- Dev: call `tenant_seed::seed_tenant_catalog(...)` from `dev_seed::run` per demo tenant (belen,
  lapaz), replacing/extending the ad-hoc `ensure_test_warehouses`.
- Prod onboarding (only if OPEN QUESTION resolves to "production"): call it from the tenant-create
  path — do NOT gate on `APP_ENV`.

## Edge cases
- Idempotency: re-running must not duplicate rows or error (ON CONFLICT DO NOTHING everywhere).
- `medical_packages.created_by` / `products` need an existing seeder user for the tenant — pass
  `seeder_id`; fail loudly if absent rather than inserting nulls.
- `package_type` enum values must match the DB enum — verify against `001_schema.sql` before use.
- Keep data catalog small/representative; do NOT port the full 936KB reference seed.

## Verification
- `cargo build`.
- Boot dev → new tenant has products, a warehouse/storage, inventory items, and ≥1 medical package
  with items. Re-boot → no duplicates, no errors.

SKILL_RECOMENDADA: none.
