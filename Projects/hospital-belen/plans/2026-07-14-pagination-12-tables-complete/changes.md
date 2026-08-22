# Changes — Pagination Full (Stage-2, effort=high)

## Backend

### Batch 0 — Infrastructure

**`hospital-belen-api/src/infrastructure/database/base.rs`**
- Added `pub use` for `Func` from sea-query
- Made `LIST_MAX_LIMIT` public
- Added `count<MC, F>(ctx, db, filters)` generic function — mirrors `list()` conditions (tenant scope, filters, soft-delete) but emits `SELECT COUNT(*)`

**`hospital-belen-api/src/web/dto/pagination.rs`** ← NEW
- `PageQuery { page: i64, limit: i64 }` with `clamped()` and `to_list_options()` helpers
- `Paginated<T: Serialize> { data, total, page, limit }` — canonical response envelope
- Constants: `PAGE_DEFAULT_LIMIT = 15`, `PAGE_MAX_LIMIT = 100`

**`hospital-belen-api/src/web/dto/mod.rs`**
- Added `pub mod pagination;`

### Batch 1 — Admin/RBAC

**`hospital-belen-api/src/domain/rbac/permission.rs`** — added `count(ctx, filter)` to trait
**`hospital-belen-api/src/domain/rbac/role.rs`** — added `count(ctx, filter)` to trait
**`hospital-belen-api/src/domain/rbac/endpoint.rs`** — added `count(ctx, filter)` to trait
**`hospital-belen-api/src/domain/rbac/endpoint_group.rs`** — added `count(ctx, filter)` to trait
**`hospital-belen-api/src/domain/menu/mod.rs`** — added `count(ctx, filter)` to trait
**`hospital-belen-api/src/domain/audit_log/mod.rs`** — added `count(ctx, filter)` to trait
**`hospital-belen-api/src/domain/tenant/mod.rs`** — added `list_paged(page, limit)` (no Ctx — platform-scoped)
**`hospital-belen-api/src/domain/user/mod.rs`** — added `count(ctx, filter)` to trait

**All repo impls updated** to implement the new count/list_paged methods.

**`hospital-belen-api/src/web/handlers/rbac.rs`** — `list_permissions`, `list_roles`, `list_endpoints` → `PageQuery` + `tokio::try_join!` + return `Paginated<T>`
**`hospital-belen-api/src/web/handlers/endpoint_group.rs`** — same
**`hospital-belen-api/src/web/handlers/menu_item.rs`** — same
**`hospital-belen-api/src/web/handlers/audit_log.rs`** — same
**`hospital-belen-api/src/web/handlers/tenant.rs`** — same
**`hospital-belen-api/src/web/handlers/user.rs`** — same (via `UserService` + `user_repo.count`)

### Batch 2 — Inventory

**`hospital-belen-api/src/domain/inventory/mod.rs`** — added `list_products_paged`, `list_inventory_paged`, `list_movements_paged` returning `(Vec<T>, i64)`
**`hospital-belen-api/src/infrastructure/database/repositories/inventory_repository.rs`** — implemented paged variants with LIMIT/OFFSET + separate COUNT queries
**`hospital-belen-api/src/web/handlers/inventory.rs`** — `list_products`, `list_inventory`, `list_movements` now return `Paginated<T>`

### Batch 3 — Clinical/Billing

**Receipts**
- `hospital-belen-api/src/web/handlers/receipt.rs` — normalized envelope from `{data, pagination:{...}}` to canonical `Paginated<T>` = `{data, total, page, limit}`

**Medical Results**
- `hospital-belen-api/src/domain/medical_result/mod.rs` — added `list_paged`
- `hospital-belen-api/src/infrastructure/database/repositories/medical_result_repository.rs` — implemented
- `hospital-belen-api/src/web/handlers/medical_result.rs` — `list_results` → `Paginated<MedicalResult>`

**Packages**
- `hospital-belen-api/src/domain/packages/mod.rs` — added `list_paged`
- `hospital-belen-api/src/infrastructure/database/repositories/package_repository.rs` — implemented
- `hospital-belen-api/src/web/handlers/packages.rs` — `list_packages` → `Paginated<MedicalPackage>`

**Doctors**
- `hospital-belen-api/src/domain/doctor/mod.rs` — added `list_paged`
- `hospital-belen-api/src/infrastructure/database/repositories/doctor_repository.rs` — implemented
- `hospital-belen-api/src/web/handlers/doctor.rs` — `list_doctors` → `Paginated<Doctor>`

### Batch 4 — Verify Already-Paginated

**Patients** — `{data, total, page, limit}` ✅
**Admissions** — `{data, total, page, limit}` ✅
**Appointments** — had `{data, total, page}`, added `limit` to response ✅

---

## Frontend

**`hospital-belen-web/kairosaid/src/types/index.ts`**
- Added `Paginated<T> { data: T[], total, page, limit }` — canonical frontend type

**`hospital-belen-web/kairosaid/src/components/common/DataTablePager.vue`** — created (was in batch 0 plan, used by UserListPage now)

**`hospital-belen-web/kairosaid/src/modules/admin/services/adminService.ts`**
- `listUsers(page, limit)` → `Paginated<AppUser>`
- `listRoles(page, limit)` → `Paginated<Role>`
- `listTenants(page, limit)` → `Paginated<TenantSummary>`

**`hospital-belen-web/kairosaid/src/modules/admin/pages/UserListPage.vue`**
- Tracks `currentPage`, `totalUsers`
- Calls `listUsers(page, limit)` → `res.data.data`
- Renders `<DataTablePager>` for prev/next navigation

**`hospital-belen-web/kairosaid/src/modules/inventory/services/inventoryService.ts`**
- `listProducts`, `listInventory`, `listMovements` → `Paginated<T>`

**`hospital-belen-web/kairosaid/src/modules/appointments/services/doctor.service.ts`**
- `list()` → `Paginated<Doctor>`

**`hospital-belen-web/kairosaid/src/modules/packages/services/packageService.ts`**
- `list()` → `Paginated<MedicalPackage>`

**`hospital-belen-web/kairosaid/src/modules/labs/services/labService.ts`**
- `list()` → `Paginated<MedicalResult>`

**`hospital-belen-web/kairosaid/src/modules/receipts/services/receiptService.ts`**
- `list()` → `Paginated<Receipt>` (updated from `PaginatedResponse`)

**`hospital-belen-web/kairosaid/src/modules/receipts/store/receiptStore.ts`**
- Reads `response.data.total/page/limit` instead of `response.data.pagination.*`
- `totalPages` now computed from `total / limit`

---

---

## Fix — Truncado silencioso + dropdowns (DISPATCH fix)

### Dropdown callers fixed (limit=100)

- `AdmissionDetailPage.vue` — `inventoryService.listInventory` autocomplete → `page:1, limit:100`
- `AdmissionDetailPage.vue` — `packageService.list` packages dropdown → `limit:100`
- `PackageDetailPage.vue` — `inventoryService.listProducts` products dropdown → `page:1, limit:100`
- `StepItems.vue` — `inventoryService.listInventory` + `packageService.list` → `page:1, limit:100`
- `InventoryMovementPage.vue` — `inventoryService.listProducts` products dropdown → `page:1, limit:100`
- `doctor.store.ts` — `doctorService.list` always → `limit:100`

### Pager added to remaining list pages

- `PermissionsPage.vue` — `DataTablePager` + `/permissions?page&limit`
- `EndpointListPage.vue` — `DataTablePager` + `/endpoints?page&limit`
- `MenuItemListPage.vue` — `DataTablePager` + `/menu-items?page&limit`
- `RoleListPage.vue` — `DataTablePager` + `adminService.listRoles(page, limit)`
- `TenantListPage.vue` — `DataTablePager` + `adminService.listTenants(page, limit)`
- `DoctorListPage.vue` — `DataTablePager` + `/doctors?page&limit`
- `PackageListPage.vue` — `DataTablePager` + `packageService.list({ page, limit })`
- `InventoryItemListPage.vue` — already done (previous)
- `ProductListPage.vue` — already done (previous)
- `InventoryMovementPage.vue` — already done (previous)
- `AuditLogPage.vue` — already had manual pager ✅
- `LabResultListPage.vue` — `DataTablePager` + `useLabResults` wired con `page`/`limit`

---

## Tester: What to verify

1. **Cargo check passes** — no compilation errors (confirmed: 0 errors, 70 warnings)
2. **Default limit** — all list endpoints default to 15 items per page
3. **Envelope** — all list endpoints return `{data: [...], total: N, page: N, limit: 15}`
4. **Receipts** — old `{data, pagination:{...}}` shape is gone; now canonical
5. **Admissions** — response now includes `limit` field alongside `total` and `page`
6. **UserListPage** — shows first 15 users + DataTablePager appears when >15 users
7. **No regressions** on: patient list, appointment list (pagination was there before)
8. **Dropdown/select usages** — places that call `listProducts()`, `listDoctors()` etc. for dropdowns still get `res.data.data` — verify dropdowns populate correctly
9. **Fix** — All admin pages (permissions, endpoints, menu-items, roles, tenants, doctors, packages) show pager when >15 records
10. **Fix** — Dropdowns in AdmissionDetailPage, PackageDetailPage, StepItems, InventoryMovementPage populate fully (100 items max, no truncation)
