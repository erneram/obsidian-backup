# Spec — PR #19 CI fix (round 2 of same clippy lint)

Branch: `fenix/modules-i18n-admissions-cleanup` @ `f8b70f3` (PR #19). Fix in place, no rebase.

## What's failing now (run 30740123473, head f8b70f3)
Still the **`check` (clippy)** job; Build+Test still skipped behind it. **Same lint as last
round, incompletely fixed.** The previous commit (`f8b70f3`) removed `&` from the two
multi-line `format!` args (old lines 42, 128) but LEFT the three inline ones. 3 errors remain,
all `useless_borrows_in_formatting` in `tests/user_soft_delete_filtering.rs`:

```
--> tests/user_soft_delete_filtering.rs:29:77   .get(format!("...{}/users", api_url(), &tenant_id))
--> tests/user_soft_delete_filtering.rs:52:77   (same line, 2nd occurrence)
--> tests/user_soft_delete_filtering.rs:138:77  (same line, 3rd occurrence)
could not compile `app` (test "user_soft_delete_filtering") due to 3 previous errors
```

Confirmed in the current file — all three still read `..., common::api_url(), &tenant_id))`.

## Fix (exact)
In `tests/user_soft_delete_filtering.rs`, at lines **29, 52, 138**, drop the `&`:
```
.get(format!("{}/api/platform/tenants/{}/users", common::api_url(), &tenant_id))
                                                                     ^ remove
→ .get(format!("{}/api/platform/tenants/{}/users", common::api_url(), tenant_id))
```
Leave line 26 `provision_user_in_tenant(&cookie, &tenant_id)` — that `&tenant_id` is a
function argument (borrow is needed), not a `format!` arg; clippy does not flag it.

## Why this recurred — do this so it's the LAST round
The fix was applied by eyeball, not by running the gate, so the inline occurrences were missed.
**Before pushing, run the actual CI gate locally and fix every site it reports:**
```bash
cd hospital-belen-api
cargo clippy --all-targets -- -D warnings   # must exit 0 — do not push until it does
```
clippy aborts at the first crate with errors, so it may surface further sites in other test
crates once this file is clean. Fixing "the 3 the log showed" is not enough — the gate passing
locally is the bar. (Memory `hospital-belen-ci-clippy-gate`: the tester skips this; whoever
pushes must run it.)

## After check goes green
Build + Test run for the first time this branch iteration. Test carries the round-5 A/B/C
fixes; its outcome is unknown until check passes. Don't touch Test now.

## Files
- `hospital-belen-api/tests/user_soft_delete_filtering.rs` (remove 3 remaining `&`)

## Notes
- No rebase/cherry-pick. No `src/` code. No new deps. branch.md unchanged.
