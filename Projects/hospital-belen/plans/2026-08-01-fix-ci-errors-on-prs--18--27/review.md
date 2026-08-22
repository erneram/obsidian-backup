VERDICT: SHIP

## Razonamiento
- Two CI-only fixes, both minimal and verified. No product logic changed.
- Web: prettier --write on PlatformSeedsPage.vue → format:check clean (verified locally).
- API: APP_DEFAULT_TENANT_ID="10000000-0000-0000-0000-000000000001" in test job env. Value matches the primary tenant + tenant_admin role seeded in migrations/002_seed.sql (verified). config.rs:136 defaults to nil UUID without the env var — that was the root cause of the fresh-DB admin 401.
- Fix A (env var) chosen over B (code) — correct call, no clippy gate needed (YAML only).

## Hallazgos
🟢 BAJO: API fix is CI-only; the fresh-DB admin path is only exercised by CI (no local repro run). CI on the branch is the real verification gate.
🟢 BAJO: Fix A leaves the nil-default in config.rs:136 unaddressed for any future fresh-DB boot without the env var — spec's fallback B would fix it permanently. Not blocking.

## PRs
API: https://github.com/InkSight-Developments/hospital-belen-api/pull/18
WEB: https://github.com/InkSight-Developments/hospital-belen-web/pull/27
MAIN: https://github.com/InkSight-Developments/hospital-belen/pull/18
