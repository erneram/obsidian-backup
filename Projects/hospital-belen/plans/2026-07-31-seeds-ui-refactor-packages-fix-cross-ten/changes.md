# Changes — fenix/seeds-ui-and-cross-tenant-packages

## A. Seeds page UI (hospital-belen-web)

**File:** `kairosaid/src/modules/platform/pages/PlatformSeedsPage.vue`

- Extended `@inksightdev/ui` import with `Select, SelectTrigger, SelectValue, SelectContent, SelectItem, Input, Label`.
- Replaced raw `<select>` tenant selector with `<Select>` block matching `PlatformRolesPage.vue` pattern; handler fires via `@update:model-value="onTenantChange"`.
- Replaced raw `<input>` name field in capture modal with `<Input>`.
- Replaced raw `<select>` module + visibility selects in capture modal with `<Select>` components.
- Labels wrapped in `<Label>`.

## B. Cross-tenant package seeding fix (hospital-belen-api)

**File:** `src/infrastructure/tenant_seed.rs`

- `capture_catalog`: added `need_storages = include_inventory || include_packages`.
- Warehouse + storage capture block now gated on `need_storages` instead of `include_inventory`.
- Result: a `packages` snapshot now carries its warehouses/storages, so `apply_catalog` can resolve `storage_code` on a fresh tenant.

## C. Builtin belen_surgery seed (hospital-belen-api)

**Files:**
- `assets/seeds/belen_surgery.json` — 567 KB JSON asset (112 products, 4 warehouses, 4 storages, 116 packages, 2726 package_items; codes only, no tenant-specific ids).
- `src/infrastructure/tenant_seed.rs` — added `BuiltinSeed { id: "packages.belen_surgery", module: "full", marker_package: Some("ABDOMINOPLASTIA-COLEC"), catalog: build_belen_surgery }` to `SEEDS`; added `build_belen_surgery()` using `include_str!`.

## Repos tocados

- `web` (hospital-belen-web)
- `api` (hospital-belen-api)

## What the Tester should review

- **A**: Select components render; tenant dropdown fires `onTenantChange`; capture modal module/visibility selects work; `selectedTenantId` empty string still hides catalog.
- **B**: Capture a `packages` snapshot; assert payload JSON has non-empty `warehouses` and `storages`. Apply to fresh tenant; assert `package_items` count > 0.
- **C**: Deserialize `belen_surgery.json` into `SeedCatalog`; assert counts (112/116/2726/4/4). Apply `builtin.packages.belen_surgery` to fresh tenant; assert products=112, medical_packages=116, package_items=2726 and `applied_seeds` returns the seed id.
