# Changes — hospital-belen Fix A (CI fixes for #18 / #27)

## hospital-belen-web

- `kairosaid/src/modules/platform/pages/PlatformSeedsPage.vue` — `prettier --write`; no logic changes. Fixes `format:check` CI step on PR #27.

## hospital-belen-api

- `.github/workflows/ci-backend.yml` — added `APP_DEFAULT_TENANT_ID: "10000000-0000-0000-0000-000000000001"` to `Test` job env. Fixes `dev_seed::ensure_admin` resolving nil tenant on fresh CI DB → admin never created → all integration tests 401. PR #18.

## Repos tocados

- `web` — 1 commit
- `api` — 1 commit

## Tester notes

- `format:check` (prettier) passes locally — verified.
- API fix is CI-only; local dev unaffected (existing DB already has admin).
- No logic changes in either repo — no new unit tests required.
- Integration smoke: boot API against fresh DB with `APP_DEFAULT_TENANT_ID` set, confirm `cargo test --all-targets` passes `admission_update` tests.
