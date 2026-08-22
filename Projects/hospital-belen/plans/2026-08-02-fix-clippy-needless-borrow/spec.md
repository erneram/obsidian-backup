# Spec — PR #19 CI fix (current branch state, no rebase/cherry-pick)

Branch: `fenix/modules-i18n-admissions-cleanup` @ `2c532cb` (PR #19). Fix in place on this branch.

## What's actually failing (latest run 30738951747)
Only the **`Type Check & Lint` (check)** job fails. `Build` and `Test` show `skipping`
because both `needs: check` — so the shared-state test fixes (A/B/C from round 5) have NOT
even run this round; they're blocked behind this clippy error.

**Cause:** the round-5 test refactor passes `&tenant_id` into `format!` args. Under
`RUSTFLAGS: -D warnings`, clippy `useless_borrows_in_formatting` makes each a hard error.
5 errors, all in `tests/user_soft_delete_filtering.rs`, all identical:

```
error: redundant reference in `format!` argument
  --> tests/user_soft_delete_filtering.rs:29 / 42 / 52 / 128 / 138
  help: remove the redundant `&`: `tenant_id`
error: could not compile `app` (test "user_soft_delete_filtering") due to 5 previous errors
```

## Fix (exact — the whole check-job failure)
In `hospital-belen-api/tests/user_soft_delete_filtering.rs`, drop the `&` before `tenant_id`
in the 5 `format!` args:
- line 29: `..., common::api_url(), &tenant_id)` → `tenant_id`
- line 42: `&tenant_id,` → `tenant_id,`
- line 52: `..., common::api_url(), &tenant_id)` → `tenant_id`
- line 128: `&tenant_id,` → `tenant_id,`
- line 138: `..., common::api_url(), &tenant_id)` → `tenant_id`

Leave the other borrows alone — they are NOT redundant and NOT format! args:
`user_soft_delete_filtering.rs:73` (`Some(&user_id)` for an `Option<&str>` compare) and
`permissions.rs:100/103/106` (`.get(&url)` reqwest arg). clippy's lint only fires on
formatting-macro args, so those are fine.

## Scope check
Verified: these 5 are the only redundant-borrow-in-format! sites across `tests/*.rs`.
clippy aborted the crate build at this test, so run the full gate locally to confirm nothing
downstream is hiding:
```bash
cd hospital-belen-api
cargo clippy --all-targets -- -D warnings   # must be clean (this is the CI gate)
cargo fmt --all -- --check
```
(See memory `hospital-belen-ci-clippy-gate` — the tester skips this gate; run it before push.)

## After check goes green
`Build` and `Test` will run for the first time this round. The Test job carries the round-5
A/B/C fixes (`--no-fail-fast`, self-provisioned fixtures, `#[serial]`); its result is unknown
until check passes. If Test then reports failures, that's a follow-up round — do NOT
pre-emptively change it now. This spec fixes only the current blocker.

## Files
- `hospital-belen-api/tests/user_soft_delete_filtering.rs` (remove 5 redundant `&`)

## Notes
- No rebasing/cherry-picking (per dispatch). Straight edit on `fenix/modules-i18n-admissions-cleanup`.
- No `src/` product code. No new deps.
- branch.md unchanged.
