# Changes — fenix/seeds-ui-and-cross-tenant-packages

## A. Seeds page UI (hospital-belen-web)

**File:** `kairosaid/src/modules/platform/pages/PlatformSeedsPage.vue`

- Extended `@inksightdev/ui` import with `Select, SelectTrigger, SelectValue, SelectContent, SelectItem, Input, Label`.
- Replaced raw `<select>` tenant selector with `<Select>` matching `PlatformRolesPage.vue` pattern; handler fires via `@update:model-value="onTenantChange"`.
- Replaced raw `<input>` name field in capture modal with `<Input>`.
- Replaced raw `<select>` module + visibility selects with `<Select>` components.
- Labels wrapped in `<Label>`.

## B. Cross-tenant package seeding fix (hospital-belen-api)

**File:** `src/infrastructure/tenant_seed.rs`

- `capture_catalog`: added `need_storages = include_inventory || include_packages`.
- Warehouse + storage capture now gated on `need_storages`; packages snapshots carry their storages.

## C. Builtin belen_surgery seed (hospital-belen-api)

**Files:**
- `assets/seeds/belen_surgery.json` — 112 products / 4 warehouses / 4 storages / 116 packages / 2726 package_items.
- `src/infrastructure/tenant_seed.rs` — `BuiltinSeed { id: "packages.belen_surgery", module: "full", marker_package: Some("ABDOMINOPLASTIA-COLEC") }` added to `SEEDS`; `build_belen_surgery()` uses `include_str!`.

## D1. Frontend CI install flag (hospital-belen-web)

**File:** `.github/workflows/ci-frontend.yml`

- All three `npm ci` lines (typecheck, lint, build jobs) changed to `npm ci --legacy-peer-deps`.

## D2. Backend test job (hospital-belen-api)

**File:** `.github/workflows/ci-backend.yml`

- Added `test` job (`needs: check`, parallel to `build`) with postgres service, API server boot via `cargo run --release &`, health wait loop on `/health`, then `cargo test --all-targets`.
- Health endpoint confirmed as `GET /health` (`src/web/router.rs:72`).
- `APP_ENV=development` avoids encryption key requirement; `SERVICE_DB_URL` + `TEST_DATABASE_URL` both point at CI Postgres.

## Repos tocados

- `web` (hospital-belen-web)
- `api` (hospital-belen-api)

## What the Tester should review

- **A**: Select components render; tenant dropdown fires `onTenantChange`; capture modal selects work.
- **B**: packages snapshot payload contains warehouses/storages; fresh-tenant apply produces package_items.
- **C**: Deserialize `belen_surgery.json` — assert 112/116/2726/4/4; apply to fresh tenant and check counts + `applied_seeds`.
- **D1**: Frontend CI runs green on PR (no peer-dep install error).
- **D2**: Test job boots server, runs `cargo test --all-targets` against live DB; build job still independent.
