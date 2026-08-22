# Spec — Fix CI on PR #18 (api) & PR #27 (web)

Branch (both repos, reuse existing PR branch): `fenix/seeds-ui-and-cross-tenant-packages`.
These are fixes to the two open PRs — commit to that same branch, do NOT open new PRs.

Two independent CI failures. Fix both.

---

## 1. hospital-belen-web — PR #27 — `Lint & Format` fails

**Symptom:** `npm run lint` passes (0 errors, warnings only). `npm run format:check`
(`prettier --check .`) fails → exit 1.

**Failing file (only one):**
- `kairosaid/src/modules/platform/pages/PlatformSeedsPage.vue` — prettier style issues.

**Fix (exact):**
```bash
cd hospital-belen-web
npx prettier --write kairosaid/src/modules/platform/pages/PlatformSeedsPage.vue
```
Then confirm `npm run format:check` and `npm run lint` are clean. No logic changes.
Do not touch the vue/max-attributes-per-line warnings — they are warnings, not errors,
and don't fail CI.

---

## 2. hospital-belen-api — PR #18 — `Test` job fails

**Symptom:** `tests/admission_update.rs` — all 3 tests FAIL at
`common::login("admin@inksight.com", "222222")` with `assertion left == right failed:
login failed for admin@inksight.com` (login returns non-200). admission_update is the
first integration binary; cargo fail-fast stops the rest, so it masks the whole suite.

**Root cause (confirmed, CI-only / fresh-DB only):**
- CI boots the server against an empty Postgres. `dev_seed::ensure_admin`
  (`src/infrastructure/dev_seed.rs:112`) takes the *create* path on a fresh DB.
- Create path looks up the role via
  `role_repo.find_by_slug("tenant_admin", cfg.default_tenant_id)` (line ~132) and
  creates the user with `tenant_id: cfg.default_tenant_id` (line ~150).
- `cfg.default_tenant_id` = env `APP_DEFAULT_TENANT_ID`, else **nil UUID**
  (`src/config.rs:133`). The CI `Test` job does **not** set `APP_DEFAULT_TENANT_ID`
  → nil.
- Migrations seed the `tenant_admin` role and the primary tenant under
  `10000000-0000-0000-0000-000000000001` (`migrations/002_seed.sql:16,98`), **not** nil.
- So `find_by_slug("tenant_admin", nil)` → `None` → `ensure_admin` logs a warn and
  returns `Ok(None)`. **admin@inksight.com is never created**, no error propagates,
  server boots healthy → every admin login 401s.
- Works on dev machines only because they reuse an existing DB where admin@inksight.com
  already exists (the *existing-user* branch at line ~125, which ignores the role lookup).

### Recommended fix — A (minimal, one line in CI)
Set the default tenant id in the `Test` job env so seeding uses the real primary tenant:

`.github/workflows/ci-backend.yml`, `Test` job `env:` block — add:
```yaml
      APP_DEFAULT_TENANT_ID: "10000000-0000-0000-0000-000000000001"
```
This makes the create path resolve the seeded `tenant_admin` role and create the admin
under the matching tenant → login works. Aligns with the existing `default_tenant_id`
config knob (it is meant to point at the primary tenant).

### Fallback fix — B (self-contained code, if the human prefers no env coupling)
In `ensure_admin`, stop trusting `cfg.default_tenant_id`; resolve the primary tenant from
the DB and use it for BOTH the role lookup and `UserForCreate.tenant_id`:
```rust
let (tenant_id,): (Uuid,) = sqlx::query_as(
    "SELECT id FROM tenants WHERE slug = 'hospital-belen' LIMIT 1",
).fetch_one(db).await.map_err(...)?;
```
Then `find_by_slug("tenant_admin", tenant_id)` and `UserForCreate { tenant_id, .. }`.
Fixes fresh-DB seeding permanently, no env dependency.

**OPEN QUESTION:** A or B? A is the shortest correct diff and is the recommendation.
Pick B only if you want the create-path to work without any env var set.

### Verify (either fix)
Reproduce a fresh-DB run the way CI does:
```bash
cd hospital-belen-api
# fresh postgres, APP_ENV=development, boot ./target/debug/app, then:
cargo test --all-targets
```
`admission_update` (and auth_login) must pass. Do not weaken the tests.

---

## Files to change
- `hospital-belen-web/kairosaid/src/modules/platform/pages/PlatformSeedsPage.vue` (prettier --write)
- `hospital-belen-api/.github/workflows/ci-backend.yml` (fix A) **or**
  `hospital-belen-api/src/infrastructure/dev_seed.rs` (fix B)

## Notes
- Both commits go to `fenix/seeds-ui-and-cross-tenant-packages` in their respective repos.
- No new PRs; these update the existing PRs #18 / #27.
- CI clippy gate: run `cargo clippy --all-targets -- -D warnings` locally before pushing
  if touching dev_seed.rs (fix B). See memory `hospital-belen-ci-clippy-gate`.
