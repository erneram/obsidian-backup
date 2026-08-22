# Spec — COMPREHENSIVE test-suite hardening (API PR #19)

Branch: PR #19 tracks **`fenix/modules-i18n-admissions-cleanup`** @ `faecc7b` — commit here
(that is the branch CI runs). Dispatch said "reuse fenix/seeds branch"; PR #19 is NOT on the
seeds branch. **OPEN QUESTION:** confirm the target branch — I'm speccing for the PR #19 branch
since that's the red CI. If you really want it on the seeds branch, say so. branch.md unchanged.

Goal: stop the one-per-round cycle. Fix EVERY remaining instance of the bug classes at once, not
just the latest red test. `list_catalog_groups_by_module` is already fixed in `faecc7b`.

## Audit result (whole suite, tests/*.rs)
I swept all 11 test files for the flagged classes — first-row indexing, pagination, JSON shape —
plus the anti-pattern that actually causes "green then red": conditional-assertion guards.

| Class | Status | Count |
|-------|--------|-------|
| JSON shape (`data.id`, array vs object, `version` from list, `error.error.type`) | ✅ fixed in recent commits; **0 remaining** | 0 |
| Pagination lookups | ⚠️ helper `find_tenant_id_by_slug` uses `?limit=100`; residual risk if ci-* count > 100 | 1 helper |
| **P1 — `["data"][0]` first-tenant indexing** | ❌ remaining | 6 |
| **P4 — conditional guards that skip assertions** | ❌ remaining (root of the flakiness) | ~39 |

## P4 — conditional guards (THE systemic bug — do this first)
Pattern everywhere:
```rust
if let Ok(resp) = resp {
    if resp.status() == 201 {
        // assertions live here
    }
}
```
When the request errors OR returns a non-201, the assertions are **skipped and the test passes
falsely**. This is exactly why #19 got a SHIP and then went red: each test passes when its happy
path isn't hit and fails when it is → nondeterministic, and negative cases are never verified.

**Fix (mechanical, every site):** unwrap the request, assert the status explicitly, then assert
outcomes — unconditionally:
```rust
let resp = resp.expect("POST /…/snapshots request failed");
assert_eq!(resp.status(), 201, "capture snapshot: {}", resp.text().await.unwrap_or_default());
let snapshot: serde_json::Value = resp.json().await.unwrap();
let id = snapshot["data"]["id"].as_str().expect("no snapshot id");
// … outcome asserts …
```
Negative tests assert the exact expected code (`assert_eq!(status, 403)` / `409`). Full checklist
of guard sites to convert:
- `tests/seed_sharing.rs`: 46, 65, 115, 134, 168, 187, 203, 243, 262, 310, 329, 365, 384, 464
- `tests/seed_snapshots.rs`: 41/42, 79/80, 98, 136/137, 154, 187/188, 225/226, 282/283
- `tests/tenant_deletion.rs`: 53, 86, 154
- `tests/tenant_seeding.rs`: 49/50, 99/100, 124, 171/172, 214/215
Where the `if` guards a genuinely-optional path (endpoint may legitimately not exist yet), keep
it but leave a `// ponytail: <reason>` — do not leave a silent skip without a note.

## P1 — `["data"][0]` first-tenant indexing (6 sites)
Grabs the arbitrary first tenant on page 1 — with `ci-*` provisioning pollution this is a random,
often content-less tenant, so product/seed/user assertions are meaningless (and today they only
"pass" because a P4 guard skips them).
- `tests/tenant_seeding.rs`: 32, 80, 155, 201  (these want hospital-belen's seeded products/seeds)
- `tests/platform_tenant_user_delete.rs`: 97
- `tests/user_soft_delete_filtering.rs`: 102

**Fix:** target a known tenant with real data via the existing helper —
`common::find_tenant_id_by_slug(&cookie, "hospital-belen").await.expect("hospital-belen not found")`
— or, for the delete/soft-delete tests, provision their own tenant + user and act on that. Never
index `["data"][0]`.

## P2 residual — harden `find_tenant_id_by_slug` (tests/common/mod.rs:145)
It fetches `?limit=100` and scans. A full `--no-fail-fast` run creates many `ci-*` tenants in the
shared DB; if the count exceeds 100 the seeded tenant falls off and lookups fail intermittently.
Pick one:
- (a) server-side filter if supported: `GET /api/platform/tenants?slug=<slug>` (check the handler);
- (b) paginate through all pages in the helper until found;
- (c) have each provisioning test delete its tenant in teardown to bound the count.
Recommend (a) if the API supports it, else (b).

## Fix strategy / order
1. P4 sweep — convert every guard to unconditional assert (biggest win; makes suite deterministic).
2. P1 — replace all 6 `["data"][0]` with slug lookup / self-provision.
3. P2 — harden the slug helper.
4. Re-run locally until stable.

## Verify (acceptance — also the loop's gate)
```bash
cargo clippy --all-targets -- -D warnings && cargo fmt --all -- --check
cargo test --all-targets --no-fail-fast     # GREEN and STABLE across 3 back-to-back runs
```
Because P4 was masking nondeterminism, the 3× stability check is mandatory — a single green run
proves nothing here.

## Files
- `tests/seed_sharing.rs`, `tests/seed_snapshots.rs`, `tests/tenant_seeding.rs`,
  `tests/tenant_deletion.rs`, `tests/platform_tenant_user_delete.rs`,
  `tests/user_soft_delete_filtering.rs`, `tests/common/mod.rs`

## Notes
- No rebase/cherry-pick. **No `src/` product code** — all defects are in tests. No new deps.
- Run all three check gates locally before push (check/clippy/fmt have each burned a round).
