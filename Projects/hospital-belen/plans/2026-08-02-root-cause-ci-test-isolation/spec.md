# Spec — Root cause: recurring CI backend `Test` job failures

**Deliverable:** diagnosis + lasting fix. This is analysis, not a single-line patch.

## TL;DR
It is **not one bug**. There are two already-patched **CI-config gaps**, one **process/git
problem** that keeps PR #18 red, and — the real recurring cause — a **test-infrastructure
design flaw**: all integration tests share one server + one persistent DB, run in parallel,
mutate rows they don't own, and depend on `data[0]` fixtures. Cargo's default **fail-fast**
surfaces exactly one failing binary per run, so every fix reveals the next latent failure →
"4 rounds and counting." The **app itself is not at fault**.

---

## Evidence

### The 4 fix rounds (all real, all necessary, none sufficient)
Commits stacked on the branch, each fixing the failure the previous one unmasked:
1. `724dd4a` — set `APP_DEFAULT_TENANT_ID` → dev_seed creates admin on fresh DB.
2. `5ee9339` — add `SERVICE_PWD_KEY`/`SERVICE_TOKEN_KEY`/`SERVICE_TOKEN_DURATION_SEC` →
   server was panicking on boot (`src/auth/config.rs:9`, `EnvVarNotFound`).
3. `0bce0f1` + `6496d07` — platform-route tests must authenticate via `platform_login`
   (platform-auth-token), not regular login.
4. `60b3319` — use a random UUID in `unauthenticated_user_cannot_delete`.

### Current state on the fully-fixed commit `60b3319`
Run 30738094494 (`fenix/modules-i18n-admissions-cleanup`, 60b3319): server boots, 5 test
binaries PASS (6/3/6/5/4), then a **new** failure appears:
`seed_sharing::recapture_snapshot_creates_new_version_immutable_ledger` panics `no id` at
`tests/seed_sharing.rs:536` — the POST that creates "versioned-snapshot" returned a body with
no `id` (creation failed), because the test reads shared `tenant_list[0]`/`[1]` and creates a
fixed-name snapshot in a DB other tests are concurrently mutating. Classic shared-state
pollution — the next domino.

### Structural facts (measured)
- `cargo test --all-targets` runs against ONE long-lived `./target/debug/app` backed by ONE
  Postgres, seeded ONCE at boot (ci-backend.yml `Test` job).
- Tests run in parallel within each binary. No `--test-threads=1`, no `serial_test`, no
  nextest, no per-test DB reset (grep: zero isolation mechanisms).
- 52 call sites read `data[0]`/first-row fixtures; 54 mutating calls (DELETE/POST/PUT) across
  the suite. Tests soft-delete users, delete tenants, recapture snapshots, share to tenants —
  all against shared global rows including the seeded admin and "first tenant."
- Cargo default is **fail-fast**: it stops after the first failing binary, hiding the rest.
  → fixes land one binary at a time = the "recurring" experience.

---

## Classification (the dispatch's question)
| Layer | Verdict |
|-------|---------|
| App issue | **No.** Boots and serves correctly once configured. |
| CI job config | **Yes, but already patched** — missing auth-key + default-tenant env vars (rounds 1-2). |
| Platform auth in tests | **Yes, partially patched** — platform_login fix (round 3); more tests still assume shared auth/fixtures. |
| Fresh-DB fixtures | **Root cause.** Tests depend on & mutate shared seed rows; no isolation. |
| Test infrastructure | **Primary recurring cause** — shared server + shared DB + parallel + fail-fast. |

---

## Compounding process problem (surface to human — important)
The fixes are **on the wrong branch relative to PR #18**:
- `origin/fenix/seeds-ui-and-cross-tenant-packages` (**PR #18**) is stuck at `724dd4a` — only
  round 1. Rounds 2-4 are NOT on it.
- `origin/fenix/modules-i18n-admissions-cleanup` carries rounds 2-4 (plus migration 012 for the
  modules task) stacked on top.

So **PR #18's own CI re-runs stale code and will stay red no matter how many fixes land on the
other branch.** Two entangled PRs are sharing one fix stack. This alone explains "keeps failing"
for anyone watching PR #18.

---

## Proposed lasting fix

### A. Stop the whack-a-mole (do first, one line) — CI config
In `ci-backend.yml` `Test` job, change the test step to:
```yaml
      - name: cargo test
        run: cargo test --all-targets --no-fail-fast
```
Surfaces ALL failing binaries in a single run instead of one per round. This ends the
"round N" cycle immediately and lets the remaining failures be fixed together.

### B. Make mutating tests self-isolating (the real fix) — test infra
Any test that MUTATES must own its fixtures; never target shared seed rows or `data[0]`:
- Create a dedicated tenant per test with a UUID-suffixed slug/name (e.g.
  `format!("test-{}", Uuid::new_v4())`), seed the minimum it needs, act on THAT, and assert on
  THAT. Do not `DELETE data[0]`, do not soft-delete the seeded admin, do not create
  fixed-name snapshots.
- Priority offenders (from CI + grep):
  - `tests/seed_sharing.rs` (recapture) — provision own source/target tenants + unique
    snapshot name; assert the create POST is 200 before reading `id`.
  - `tests/user_soft_delete_filtering.rs` — deletes `users[0]` of `tenants[0]` (can be the
    seeded admin) → poisons every later login. Create + delete its own user.
  - `tests/platform_tenant_user_delete.rs`, `tests/tenant_deletion.rs`,
    `tests/admission_update.rs` — same pattern.
- Add `assert_eq!(resp.status(), 200/201, "…: {body}")` before every `["id"]`/`["data"][0]`
  unwrap so a failed request fails with a clear message, not a downstream `expect("no id")`.

### C. Contain remaining shared state — pick one
- **Cheaper:** add `serial_test` and mark the handful of mutating tests `#[serial]`; keep
  read-only tests parallel. Removes cross-test races without full rewrite.
- **Cleaner (later):** reset+reseed the DB between binaries (drop/recreate schema or run in a
  txn rolled back), so each binary starts from a known state. Bigger lift.
Recommend B + C-serial now; C-reseed as follow-up.

### D. Branch strategy (OPEN QUESTION — human/manager decides)
The CI-config + test fixes (rounds 2-4) must reach the branch whose CI must go green.
Options:
1. Cherry-pick/rebase rounds 2-4 onto `fenix/seeds-ui-and-cross-tenant-packages` (PR #18) and
   push, so PR #18 goes green; keep migration 012 (a modules-task change) on the modules branch.
2. Merge PR #18 as-is via the modules branch and close #18 in favor of the combined branch.
3. Retarget PR #18 to the modules branch.
**OPEN QUESTION:** which branch owns the CI/test fixes, and do we keep #18 and the modules PR
separate or fold them? The Coder must not push more fixes until this is decided, or they'll keep
landing where PR #18 can't see them.

---

## Files in scope
- `hospital-belen-api/.github/workflows/ci-backend.yml` — fix A (`--no-fail-fast`).
- `hospital-belen-api/tests/seed_sharing.rs`, `tests/user_soft_delete_filtering.rs`,
  `tests/platform_tenant_user_delete.rs`, `tests/tenant_deletion.rs`,
  `tests/admission_update.rs` — fix B (self-provisioned fixtures + status asserts).
- `hospital-belen-api/Cargo.toml` + those tests — fix C-serial (`serial_test`, `#[serial]`).

## Verify
Fresh-DB run the CI way, then:
```bash
cargo test --all-targets --no-fail-fast    # must be green, and STABLE across 3 back-to-back runs
```
Stability across repeated runs is the acceptance bar — a single green run does not prove
isolation. See memory `hospital-belen-fresh-db-admin-seed`, `hospital-belen-ci-clippy-gate`.

## Notes
- `branch.md` left untouched pending the fix-D branch decision — do not assume a branch yet.
- No new dependency beyond `serial_test` (dev-dependency) if fix C-serial is chosen.
