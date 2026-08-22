# Changes — fenix/fix-platform-user-delete-500 (round 4)

## Backend (hospital-belen-api)

### `migrations/010_seed_snapshots.sql` (new)
- `seed_visibility ENUM ('PRIVATE', 'SHARED')`
- `seed_snapshots` table: id, source_tenant_id, name, module, version (auto-incremented), visibility, payload JSONB, created_by (platform_users.id), created_at
- Unique `(source_tenant_id, name, version)` — immutable versions
- Index on visibility for SHARED filter

### `src/infrastructure/tenant_seed.rs` (refactored)
- `SeedCatalog` / `ProductRow` / `WarehouseRow` / `StorageRow` / `PackageRow` / `PackageItemRow` — serde-compatible shape for both builtin const and snapshot JSONB
- `BuiltinSeed` struct + `SEEDS` static array: `builtin.inventory`, `builtin.packages.pkg_cir_001`, `builtin.packages.pkg_par_001`
- `apply_catalog(db, tenant_id, seeder_id, &SeedCatalog)` — unified insert path (idempotent)
- `apply_builtin(db, tenant_id, seeder_id, seed_id)` — looks up SEEDS, calls apply_catalog
- `apply_snapshot(db, tenant_id, seeder_id, snapshot_id)` — loads JSONB payload, enforces PRIVATE policy → DomainError::Conflict (→ 403)
- `remove_seed(db, tenant_id, seed_id)` — FK-order deletion; FK violation → DomainError::Conflict (→ 409)
- `applied_seeds(db, tenant_id)` — presence checks by marker code for builtin + snapshot seeds
- `capture_snapshot(db, source_tenant_id, module, name, visibility, created_by)` — SELECTs live catalog, serializes to JSONB, auto-increments version
- `resolve_tenant_seeder(db, tenant_id)` — finds first active user in tenant for created_by FKs
- `seed_tenant_catalog` — unchanged signature, now calls apply_catalog per SEEDS entry

### `src/web/handlers/platform_seed.rs` (new)
7 endpoints, all behind `mw_platform_ctx_require`:
- `GET  /api/platform/seeds?tenantId=` — builtin + visible snapshot seeds grouped by module
- `GET  /api/platform/tenants/{id}/seeds` — applied seed IDs
- `POST /api/platform/tenants/{id}/seeds` — apply list of seed IDs (builtin or snapshot)
- `DELETE /api/platform/tenants/{id}/seeds/{seed_id}` — remove seed rows (409 if in use)
- `POST /api/platform/tenants/{id}/snapshots` — capture (201 + {id})
- `GET  /api/platform/snapshots` — list all snapshots (admin view)
- `DELETE /api/platform/snapshots/{id}` — delete snapshot definition

### `src/web/handlers/mod.rs` + `src/web/router.rs`
- Added `pub mod platform_seed` and wired all 7 routes into the platform group

## Frontend (hospital-belen-web)

### `src/modules/platform/services/platformSeed.service.ts` (new)
- Types: `SeedEntry`, `SeedModule`, `Snapshot`, `CaptureInput`
- Methods: `listCatalog`, `appliedFor`, `apply`, `remove`, `listSnapshots`, `capture`, `deleteSnapshot`

### `src/modules/platform/pages/PlatformSeedsPage.vue` (new)
- Tenant dropdown → parallel load catalog + applied state
- Seed rows: checkbox, name, version, kind chip (built-in/snapshot), applied badge, remove button
- "Apply selected" batch action
- Capture modal: name, module select, visibility select → POST snapshot
- Snapshots admin table with delete

### `src/modules/platform/router.ts`
- Added `{ path: 'seeds', name: 'platform-seeds', component: PlatformSeedsPage }`

### `src/layouts/PlatformLayout.vue`
- Added `Database` icon import from lucide
- Added nav entry `{ to: '/platform/seeds', icon: Database, label: 'platform.nav.seeds' }`

### `src/locales/es.json`
- `platform.nav.seeds` + full `platform.seeds.*` key set

## Repos tocados
- `api` (hospital-belen-api)
- `web` (hospital-belen-web)

## What the Tester should review
- Migration 010 applies cleanly
- `cargo build` passes
- Capture tenant catalog → row in seed_snapshots with JSONB payload
- Apply snapshot to different tenant → packages/products created; PRIVATE cross-tenant → 403
- Re-apply → no duplicates
- Remove seed with product in use → 409
- Capture same name twice → version 2
- Frontend: tenant dropdown → seeds render with kind chips + applied badges; apply/remove flip badges; capture modal creates snapshot visible in table and catalog
