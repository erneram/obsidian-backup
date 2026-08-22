# Spec — PR #19 check job: now `cargo fmt` failing

Branch: `fenix/modules-i18n-admissions-cleanup` @ `9fc2a1d` (PR #19). Fix in place, no rebase.

## Answer to the dispatch's question
This is the **backend Rust repo** (`hospital-belen-api`). There is **no eslint** here — that's
a frontend concern. The `Type Check & Lint` (`check`) job runs three sub-steps in order:
`cargo check` → `cargo clippy` → `cargo fmt --all -- --check`.

Current status of the three (run 30740646531, head 9fc2a1d):
- `cargo check` — **passes**.
- `cargo clippy` — **passes** (last round's `&tenant_id` fix worked).
- `cargo fmt --all -- --check` — **FAILS**. This is the only failing sub-step now.

So: not type-check, not lint/clippy — **formatting**. The round-5 test refactor was never run
through rustfmt.

## What's unformatted
Many diffs across the round-5 test files (rustfmt wants long `.get(format!(...))`, `assert!`,
and helper calls wrapped multi-line):
- `tests/common/mod.rs` (lines ~89, 99, 106, 115 — the new `provision_*` helpers)
- `tests/user_soft_delete_filtering.rs` (~26, 49, 72, 91, 102, 135, 147)
- `tests/platform_tenant_user_delete.rs` (~109, 131)
- `tests/seed_sharing.rs` (~530)
- `tests/tenant_deletion.rs` (~24, 31, 64, 71, 89, 96, 109, 118, 140, 151, 168, 188, 195)

## Fix (mechanical — do not hand-edit)
```bash
cd hospital-belen-api
cargo fmt --all
```
Commit the result. That resolves every diff. Do not hand-format — let rustfmt do it.

## Stop the round-by-round cycle (this is round 3 of the same job)
Each round one `check` sub-step was fixed and the next surfaced (clippy → clippy again → fmt).
Before pushing, run the **full** gate locally and only push when all three exit clean:
```bash
cargo check --all-targets
cargo clippy --all-targets -- -D warnings
cargo fmt --all -- --check      # or run `cargo fmt --all` first, then this must pass
```
(Memory `hospital-belen-ci-clippy-gate`: the tester skips these; whoever pushes must run them.)

## After check goes green
Build + Test finally run for the first time this iteration. Test carries the round-5 A/B/C
fixes; outcome unknown until check passes. Don't touch Test now.

## Files
- `hospital-belen-api/tests/common/mod.rs`, `tests/user_soft_delete_filtering.rs`,
  `tests/platform_tenant_user_delete.rs`, `tests/seed_sharing.rs`, `tests/tenant_deletion.rs`
  — all resolved by `cargo fmt --all`.

## Notes
- No rebase/cherry-pick. No `src/` product code. No new deps. branch.md unchanged.
