VERDICT: SHIP

## Razonamiento
- Commit f9936dc fixes the real app bug from last round's NEEDS_WORK, correctly and minimally.
- capture_catalog (tenant_seed.rs) now casts NUMERIC → ::float8 at all three read sites:
  :800 default_price::float8, :855 base_cost::float8, :869 quantity::float8 + unit_price::float8.
  float8 = double precision → sqlx decodes into f64 cleanly (the NUMERIC→f64 mismatch is gone).
- Matches the existing inventory query pattern (per dispatch); snapshot capture now returns 201.
- Verified: these are the only query_as-f64 NUMERIC reads in src/; all now cast. fmt + clippy clean.

## Hallazgos
🟢 BAJO: the `rust_decimal` sqlx feature added last round while diagnosing is now UNUSED (no Decimal
   anywhere in src/ — the fix went the float8 route). Drop it from Cargo.toml + regen lock to shed the
   rkyv/borsh/bitvec transitive deps. Harmless, non-blocking.

## Acceptance (CI-runtime, not verifiable locally)
- The capture→apply path now decodes NUMERIC; confirm on the 3x --no-fail-fast CI run that snapshot
  capture/apply tests (seed_snapshots/seed_sharing) go green AND stable before /approve. I cannot run
  the integration suite locally (needs fresh Postgres + running server).

## PRs
API: https://github.com/InkSight-Developments/hospital-belen-api/pull/19
WEB: https://github.com/InkSight-Developments/hospital-belen-web/pull/28
MAIN: https://github.com/InkSight-Developments/hospital-belen/pull/19
