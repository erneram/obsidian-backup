# Changes — fenix/seeds-ui-and-cross-tenant-packages (CI round 2)

## F1. ESLint: camelCase Progress prop (hospital-belen-web)

**File:** `kairosaid/src/modules/platform/pages/PlatformTenantDetailPage.vue:201`

- `:model-value` → `:modelValue` on `<Progress>` component (vue/attribute-hyphenation).
- `@update:model-value` event listeners elsewhere unchanged (rule targets attributes, not events).

## F2. Backend test job: build-before-start + correct health URL + log dump (hospital-belen-api)

**File:** `.github/workflows/ci-backend.yml`

- Replaced `cargo run --release &` (cold release build inside health window) with:
  1. `cargo build --all-targets` (debug, foreground — also compiles test binaries)
  2. `./target/debug/app > server.log 2>&1 &`
- Health wait loop: 60×2s, hits `http://localhost:8080/health` (was `/`).
- On timeout: dumps `server.log` via `::error::` annotation before exit 1.

## Repos tocados

- `web` (hospital-belen-web)
- `api` (hospital-belen-api)

## What the Tester should review

- **F1**: `cd kairosaid && npm run lint` exits 0 (no vue/attribute-hyphenation error).
- **F2**: `Test` job goes green; server.log visible in run output on failure.
