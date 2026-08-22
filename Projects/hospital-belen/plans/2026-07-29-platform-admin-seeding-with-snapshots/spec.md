# Spec — Platform Admin Seeding UI, Option B: tenant-authored snapshots (round 4, revised)

RESOLVED: Interpretation **B**. Seeds are no longer only built-in static entries. A tenant's current
catalog can be **captured as a versioned snapshot**, stored in the DB, and applied to other tenants
subject to a sharing policy. The built-in static registry from `tenant_seed.rs` stays as one seed
source; snapshots are a second source. Both surface through the same catalog/apply API and UI.

Builds on `hospital-belen-api/src/infrastructure/tenant_seed.rs`. `branch.md` unchanged.

---

## Seed sources (unified model)
A "seed" is anything appliable to a tenant. Two sources, one `seed_id` namespace:
- **Built-in** — `builtin.<id>` (from `SEEDS` const, e.g. `builtin.packages.pkg_cir_001`). Static.
- **Snapshot** — `snapshot.<uuid>` — a captured, versioned catalog stored in `seed_snapshots`.

The catalog/apply/remove API treats both uniformly; only the *data origin* differs (const vs JSONB).

---

## Part A — Backend

### A1. Snapshot storage — new migration `migrations/010_seed_snapshots.sql`
(Next free number; migrator discovers `.sql` files sorted by filename — `migrator.rs`.)
```sql
CREATE TYPE seed_visibility AS ENUM ('PRIVATE', 'SHARED');

CREATE TABLE seed_snapshots (
  id               uuid            PRIMARY KEY DEFAULT gen_random_uuid(),
  source_tenant_id uuid            NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name             varchar(200)    NOT NULL,
  module           varchar(50)     NOT NULL,      -- 'inventory' | 'packages' | 'full'
  version          integer         NOT NULL DEFAULT 1,
  visibility       seed_visibility NOT NULL DEFAULT 'PRIVATE',
  payload          jsonb           NOT NULL,      -- captured rows (see A2 shape)
  created_by       uuid            NOT NULL REFERENCES platform_users(id),
  created_at       timestamptz     NOT NULL DEFAULT now(),
  UNIQUE (source_tenant_id, name, version)
);
CREATE INDEX idx_seed_snapshots_visibility ON seed_snapshots (visibility);
```
Rationale (ponytail): store the captured catalog as **one JSONB payload** per snapshot, not a
relational mirror of products/packages/items. Apply reads the payload and inserts. Avoids a parallel
schema and keeps capture/apply symmetric.

### A2. Capture — snapshot a tenant's live catalog
New fn in `tenant_seed.rs` (or `seed_snapshot.rs`):
```rust
pub async fn capture_snapshot(
    db, source_tenant_id, module, name, visibility, created_by: platform_user_id,
) -> Result<Uuid>;
```
- SELECT the tenant's current rows for `module` and serialize to the payload shape below (by **code**,
  never raw ids — ids are tenant-specific and must be re-resolved on apply):
```jsonc
{
  "products":   [{ "code","name","category","unit_abbr","default_price" }],
  "warehouses": [{ "code","name" }],
  "storages":   [{ "warehouse_code","code","name","type" }],
  "packages":   [{ "code","name","type","description","base_cost" }],
  "package_items": [{ "package_code","product_code","storage_code","quantity","unit_price","sort_order" }]
}
```
  `module='packages'` captures packages + their referenced products/storages; `inventory` captures
  products/warehouses/storages/inventory; `full` captures everything. (Keep the SELECT scoped so a
  package snapshot is self-contained and applies cleanly.)
- **Versioning**: `version = max(version)+1` for the same `(source_tenant_id, name)`; first is 1.
  Never mutate an existing snapshot row — capture always inserts a new version (immutable history).

### A3. Registry stays, apply becomes source-aware
Refactor `tenant_seed.rs` as in the prior round (built-in `SEEDS` catalog + per-seed apply), then
generalize apply to accept a payload so built-in and snapshot share one insert path:
```rust
async fn apply_catalog(db, tenant_id, seeder_id, catalog: &SeedCatalog) -> Result<()>; // core inserts
pub async fn apply_builtin(db, tenant_id, seeder_id, seed_id) -> Result<()>;  // SEEDS[seed_id] → apply_catalog
pub async fn apply_snapshot(db, tenant_id, seeder_id, snapshot_id) -> Result<()>; // payload → SeedCatalog → apply_catalog
pub async fn remove_seed(db, tenant_id, seed_id) -> Result<()>;   // deletes that seed's rows (FK-order)
pub async fn applied_seeds(db, tenant_id) -> Result<Vec<String>>; // ids present for tenant
```
- `SeedCatalog` is the deserialized payload shape (A2); built-in const rows map into the same struct.
- All inserts stay idempotent (`ON CONFLICT (tenant_id, code) DO NOTHING`, `NOT EXISTS` for
  package_items) and re-resolve ids by code within the target tenant → **cross-tenant apply works**.
- `seed_tenant_catalog` (dev seed / onboarding) = apply all built-in seeds. Unchanged callers.

### A4. Sharing policy
- `visibility='PRIVATE'` → snapshot appliable only to its `source_tenant_id`.
- `visibility='SHARED'` → appliable to any tenant.
- Built-in seeds are implicitly shared (any tenant).
- Catalog for a target tenant = all built-in + all SHARED snapshots + PRIVATE snapshots whose
  `source_tenant_id == target`. Enforce this filter **server-side** on both list and apply (apply a
  PRIVATE snapshot to a non-owner tenant → `403`).

### A5. Platform API — `src/web/handlers/platform_seed.rs` (new)
Mirror `platform_plans.rs` (`Ctx`, `Json(json!({"data":...}))`, `WebError::Domain`), behind
`mw_platform_ctx_require`. Register in `router.rs` platform group.

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/platform/seeds?tenantId=` | catalog for target tenant: built-in + visible snapshots, grouped by module. `tenantId` needed to filter PRIVATE snapshots |
| GET | `/api/platform/tenants/{tenant_id}/seeds` | `{ applied: [seedId,...] }` |
| POST | `/api/platform/tenants/{tenant_id}/seeds` | body `{ seedIds:[...] }` → apply each (built-in or snapshot); `204` |
| DELETE | `/api/platform/tenants/{tenant_id}/seeds/{seed_id}` | remove one seed's rows; `204` / `409` if in use |
| POST | `/api/platform/tenants/{tenant_id}/snapshots` | capture: body `{ name, module, visibility }` → `201 { id }` |
| GET | `/api/platform/snapshots` | list all snapshots (platform admin view: id, sourceTenant, name, module, version, visibility, createdAt) |
| DELETE | `/api/platform/snapshots/{id}` | delete a snapshot definition (not tenant data) |

- Validate `seed_id` against built-in `SEEDS` ∪ existing snapshot ids; unknown → `404`.
- Apply visibility check (A4) → `403` on PRIVATE cross-tenant.

### A6. Removal safety (unchanged from prior round)
`remove_seed` deletes only that seed's own rows in FK-dependency order (`package_items` →
`medical_packages`; `inventory_items` → `storages`/`warehouses` → `products`), by code + tenant.
Rows referenced by real tenant data (admissions, other seeds' items, movements) FK-fail → catch →
`409 Conflict "seed in use"`. Never cascade real data.

---

## Part B — Frontend (`hospital-belen-web/kairosaid`)

### B1. Files
- `src/modules/platform/services/platformSeed.service.ts` (new) — mirror `platformPlan.service.ts`.
- `src/modules/platform/pages/PlatformSeedsPage.vue` (new).
- `src/modules/platform/router.ts` — add `{ path:'seeds', name:'platform-seeds', component: () =>
  import('./pages/PlatformSeedsPage.vue'), meta:{title:'Seeds'} }`.
- `src/layouts/PlatformLayout.vue` — nav item `{ to:'/platform/seeds', icon: Database,
  label:'platform.nav.seeds' }` (import `Database` from lucide).
- `src/locales/es.json` — `platform.nav.seeds` + page strings.

### B2. Service methods
```ts
listCatalog(tenantId): Promise<SeedModule[]>        // GET /platform/seeds?tenantId
appliedFor(tenantId): Promise<string[]>             // GET /platform/tenants/:id/seeds
apply(tenantId, seedIds: string[]): Promise<Result>
remove(tenantId, seedId): Promise<Result>
listSnapshots(): Promise<Snapshot[]>                // GET /platform/snapshots
capture(tenantId, { name, module, visibility }): Promise<Result<{id}>>  // POST snapshots
deleteSnapshot(id): Promise<Result>
```
Types: `SeedModule { module; seeds:{ id; name; variant; kind:'builtin'|'snapshot'; version? }[] }`,
`Snapshot { id; sourceTenant; name; module; version; visibility; createdAt }`.

### B3. Page behaviour
1. **Tenant dropdown** — `platformTenantService.list()`. Nothing loads until chosen.
2. On select → `listCatalog(tenantId)` + `appliedFor(tenantId)` in parallel.
3. **Seed catalog** — modules as sections; each seed a row (checkbox + name + muted `variant`/version
   + a `kind` tag "built-in"/"snapshot" + applied badge).
   - Apply selected unapplied → `apply()`; remove applied → `remove()`; refresh after each; `409` →
     toast, keep checked.
4. **Capture snapshot** — a "Capture current catalog as seed" action for the selected tenant: modal
   with `name`, `module` (inventory/packages/full), `visibility` (PRIVATE/SHARED). Submit →
   `capture()` → refresh catalog (new snapshot appears).
5. **Snapshots admin** (optional section) — `listSnapshots()` table with delete.

## Diseño visual
Consistency with existing platform pages — no new visual language.
- Shell like `PlatformFeaturesPage.vue` (title, description, card).
- Tenant selector at top (existing dropdown component).
- Module cards `border rounded-lg`, `text-sm font-semibold` header; seed rows = checkbox + name +
  muted variant/version + a small `kind` chip + status badge (`bg-green-100 text-green-800` applied /
  muted not-applied, matching `PlatformTenantDetailPage.vue:33`).
- Buttons (`@inksightdev/ui`, variants default|destructive|outline|secondary|ghost|link):
  "Apply selected" = `default`; per-row remove = `ghost`/`outline`; "Capture snapshot" = `outline`;
  snapshot delete = `destructive` (deletes a definition, safe).
- Capture modal: existing dialog/modal component; plain inputs + a select for module/visibility.
- Loading via `:loading` prop; toasts via `@inksightdev/ui`.

---

## Edge cases
- **Immutable versions**: re-capturing the same name inserts a new `version`; never overwrite. UI
  shows version next to name.
- **PRIVATE cross-tenant apply** blocked server-side → `403`; UI shouldn't offer it (catalog already
  filtered), but enforce on the API regardless.
- **Snapshot payload by code, not id** — apply re-resolves ids in the target tenant; a product code
  missing in target is created from the payload row (self-contained capture guarantees this).
- **seeder_id** for apply/`created_by`: resolve the target tenant's seeder/admin user; if none,
  clear error, not an FK 500. Snapshot `created_by` = acting `platform_users.id`.
- **Idempotent apply** → 204 no-op if already applied.
- **Delete snapshot** removes only the definition row (`ON DELETE CASCADE` from tenant covers orphan
  cleanup); does NOT touch already-applied tenant rows.
- **Applied state reflects DB truth** — re-fetch after every mutation.

## Verification
- Backend: `cargo build`; migration applies; capture a tenant's packages → row in `seed_snapshots`
  with JSONB payload; apply that snapshot to a *different* tenant → its packages/products appear;
  re-apply → no dupes; apply a PRIVATE snapshot cross-tenant → 403; remove a seed whose product is in
  use → 409; capture same name twice → version 2.
- Frontend: pick tenant → built-in + snapshot seeds render; apply subset → badges flip; capture modal
  creates a snapshot that then appears (and, if SHARED, for other tenants); remove/delete flows work.

SKILL_RECOMENDADA: none.
