# Changes — fenix/fix-platform-user-delete-500 (round 3)

## P1 — Exclude INACTIVE users from list/count

### `src/infrastructure/database/base.rs`
- `list`: added `SoftSetField { column, value, cast }` exclusion block after the existing
  `SoftIsDeleted` block — emits `WHERE <column> <> <value>::<type>` with optional cast.
- `count`: same predicate added — pagination totals now match visible rows.
- `cast: None` path emits plain `<column> <> <value>`, no regression for non-enum columns.

## P2 — Modular tenant catalog seeder

### `src/infrastructure/tenant_seed.rs` (new)
- `seed_tenant_catalog(db, tenant_id, seeder_id)` — public entry point, idempotent.
- `seed_products`: ensures `hora`/`noche` units, inserts 8 baseline products (5 supplies + 3
  services) via `ON CONFLICT (tenant_id, code) DO NOTHING`.
- `seed_inventory`: inserts 2 warehouses (`SALA-OP`, `ENF`) + 2 storages (`OR`, `ENF-GEN`) +
  inventory items for supply products per storage, all idempotent.
- `seed_medical_packages`: inserts 2 packages (`PKG-CIR-001` SURGERY, `PKG-PAR-001` DELIVERY)
  + 9 package_items via `NOT EXISTS` subquery (no unique constraint on `package_items`).

### `src/infrastructure/mod.rs`
- Added `pub mod tenant_seed;`.

### `src/infrastructure/dev_seed.rs`
- Replaced `ensure_test_warehouses(db, belen_id, seeder_id)` call with
  `tenant_seed::seed_tenant_catalog(db, belen_id, seeder_id)` +
  `tenant_seed::seed_tenant_catalog(db, lapaz_id, seeder_id)`.
- Removed dead `ensure_test_warehouses` function.

## Repos tocados

- `api` (hospital-belen-api)

## What the Tester should review

### P1
- Soft-delete a user → `GET /api/platform/tenants/{t}/users` no longer lists them.
- `count` response drops by 1 to match.
- An ACTIVE user remains listed.

### P2
- Boot dev → belen and lapaz tenants each have 8 products, SALA-OP+ENF warehouses,
  OR+ENF-GEN storages, inventory items for supply products, 2 medical packages with items.
- Re-boot → no duplicates, no errors (idempotent).
- `seed_tenant_catalog` callable standalone with any `tenant_id` + valid `seeder_id`.
