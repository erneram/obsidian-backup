# Spec — PR #19 Test job: remaining failures (isolation gaps)

Branch: `fenix/modules-i18n-admissions-cleanup` @ `987697b` (PR #19). Tests-only, no rebase.

## Diagnosis (the dispatch's question)
**These are the isolation gaps the reviewer flagged — NOT new app bugs.** Zero `src/` changes;
handlers are correct. Down 15→8 (dispatch summarized as "4"; CI shows 8). Two mechanisms +
two response-shape mismatches, all in tests:

1. **Pagination + provisioning pollution.** `/api/tenants` and `/api/platform/tenants` are
   paginated (`PageQuery`/`Paginated`, `src/web/handlers/tenant.rs:25`). Round-5's
   `provision_tenant()` floods the shared DB with `ci-*` tenants, so tests that scan page-1
   `data[]` for a seeded demo tenant no longer find it.
2. **Content-less provisioned sources.** Snapshots captured from empty `ci-*` tenants have no
   packages/inventory, so apply/catalog assertions on real content fail.
3. **Capture response has no `version`.** `capture_snapshot` returns `{"data":{"id":...}}`
   only (`platform_seed.rs:231`); `version` exists only on GET `/api/platform/snapshots`
   (`platform_seed.rs:255`).
4. **Error body shape.** API errors are `{"error":{"type":...,"req_id":...}}` — `error` is an
   object, and there is no top-level `message`.

## Failures & targeted fixes (tests only)

### seed_sharing::recapture_snapshot_creates_new_version_immutable_ledger (v1=0, v2=0)
`seed_sharing.rs:474`. Reads `snap_v2["data"]["version"]`, but the capture POST returns only
`{data:{id}}` → `None` → 0 for both v1 and v2 → `assert!(v2 > v1)` fails.
**Fix:** after each capture, GET `/api/platform/snapshots`, find the row by `id` (or `name`),
read its `data[].version`, and compare those. (list_snapshots returns `version`; capture does
not.) No `src/` change.

### seed_snapshots: apply_snapshot_to_tenant_creates_catalog (95), apply_private…403 (148), recapture_same_name_increments_version (205), list_catalog_groups_by_module (292)
Apply POST `/api/platform/tenants/{target}/seeds` expected 200/403 but got other. Two likely
causes to check locally:
- **Empty source:** the snapshot is captured from a provisioned `ci-*` tenant with no
  packages/inventory. Capture the snapshot from a **seeded** tenant that has real module data
  (e.g. `hospital-belen`) as SOURCE; apply to a provisioned TARGET.
- **Id-format mismatch:** `list_seed_catalog` exposes ids as `format!("snapshot.{}", id)`
  (`platform_seed.rs:75`) while capture returns the raw UUID. Confirm which form
  `apply_seeds`/`seedIds` expects and pass that form. (`recapture_same_name` also needs the
  version-from-list fix above.)

### tenant_isolation::platform_admin_can_switch_tenant_and_see_lapaz_patients ("lapaz not found")
`tenant_isolation.rs:150`. Logs in with a **regular** `auth-token` and scans page-1 of
`/api/tenants` for `slug=="lapaz"`. With many `ci-*` tenants, `lapaz` is pushed off page 1
(and a regular login may be tenant-scoped).
**Fix:** look up `lapaz` deterministically — `platform_login` + GET `/api/platform/tenants`
with a large `limit` (PageQuery) or a slug filter, or reference the seeded lapaz id — instead
of scanning the default page. Don't depend on `lapaz` being on page 1.

### tenant_deletion::delete_default_tenant_returns_409 (91)
The 409 now works (the earlier `assert_eq!(status,409)` passes). It panics at line 91 inside
`if status==409` because it reads `error["message"]` / `error["error"].as_str()`, but the body
is `{"error":{"type":...}}` (`error` is an object, no `message`).
**Fix:** assert on `error["error"]["type"].is_string()` (or drop the message check). No `src/`.

### tenant_deletion::delete_tenant_cascade_removes_child_rows (154)
After deleting its provisioned tenant, GET `/api/platform/tenants/{id}/users` is asserted to be
404/403 but returned otherwise. Verify the real post-delete behavior and assert the actual
status (it may return 200 with an empty list, or 500). Adjust the expectation to match the app,
or assert the child rows are gone via the response body.

## Convention to close this class for good (recommend to reviewer)
Pick ONE and apply across the suite: (a) fully self-provision AND never scan the global list —
look up any needed tenant by exact id or slug filter; or (b) for seeded-tenant lookups, always
pass a high `limit`/slug filter so pagination can't hide them. The current mix of provisioned
fixtures + page-1 scans of seeded demo data is the remaining gap.

## Verify
```bash
cargo clippy --all-targets -- -D warnings && cargo fmt --all -- --check
cargo test --all-targets --no-fail-fast     # GREEN and STABLE across 3 back-to-back runs
```

## Files
- `tests/seed_sharing.rs`, `tests/seed_snapshots.rs`, `tests/tenant_isolation.rs`,
  `tests/tenant_deletion.rs`

## Notes
- No rebase/cherry-pick. **No `src/` product code.** No new deps. branch.md unchanged.
- Run all three check gates locally before push.
