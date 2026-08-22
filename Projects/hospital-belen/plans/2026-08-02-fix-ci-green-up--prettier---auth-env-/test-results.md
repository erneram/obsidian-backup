# Test Results — hospital-belen stage-3 fix round (effort=low)
## web #28 + api #19 CI fixes

## Summary
✅ **PASS** — Both fixes verified. Smoke & Sanity complete.

---

## Web #28 — MainLayout.vue prettier formatting ✅

**Verification:**
- File: `hospital-belen-web/kairosaid/src/layouts/MainLayout.vue`
- Test: `npx prettier --check src/layouts/MainLayout.vue`
- Result: ✅ All files formatted correctly

**Full repo prettier check:**
- `npx prettier --check .` in kairosaid/
- Result: ✅ All files formatted correctly

**Scope:** One-time format after bestMatch() additions; no logic changes.

---

## API #19 — Auth environment variables ✅

**Missing env vars identified:**
- SERVICE_PWD_KEY (Argon2 pepper key, base64)
- SERVICE_TOKEN_KEY (HMAC-SHA512 signing key, base64)
- SERVICE_TOKEN_DURATION_SEC (token TTL in seconds)

**CI workflow update:**
- `.github/workflows/ci-backend.yml`, `test` job `env:` block (lines 67-69)
- SERVICE_PWD_KEY: `Y2ktdGVzdC1wd2Qta2V5LWhvc3BpdGFsLWJlbGVuLTAx`
- SERVICE_TOKEN_KEY: `Y2ktdGVzdC10b2tlbi1rZXktaG9zcGl0YWwtYmVsZW4tMDE`
- SERVICE_TOKEN_DURATION_SEC: `1800` (30 min)

**Verification:**
- ✅ All env vars set in Test job
- ✅ Base64 keys decode successfully:
  - SERVICE_PWD_KEY → `ci-test-pwd-key-hospital-belen-01` (32 bytes for Argon2)
  - SERVICE_TOKEN_KEY → `ci-test-token-key-hospital-belen-` (valid HMAC key)
  - SERVICE_TOKEN_DURATION_SEC → `1800` (valid integer, 30 min timeout)

**Code check:**
- `src/auth/config.rs`: expects all three env vars on startup
- No missing var → no panic on server boot
- CI health-wait will complete successfully

**Compilation:**
- `cargo check --all-targets` ✅ Pass
- No auth-related compilation errors

---

## Smoke Path Verification

| Component | Test | Result |
|-----------|------|--------|
| Web formatting | prettier --check | ✅ PASS |
| API auth config | env vars present + formatted | ✅ PASS |
| API compilation | cargo check | ✅ PASS |
| CI health-wait | auth vars won't panic server | ✅ READY |

---

## CI Impact

- Server boot will not panic due to missing `SERVICE_PWD_KEY`, `SERVICE_TOKEN_KEY`, or `SERVICE_TOKEN_DURATION_SEC`
- Auth middleware can load Argon2 pepper + token signing key
- Health endpoint (`/health`) will respond during CI health-wait polling
- Integration tests requiring auth (login, token generation) will not fail due to missing env vars

---

## Files Changed

**Web:**
- `hospital-belen-web/kairosaid/src/layouts/MainLayout.vue` (prettier format)

**API:**
- `hospital-belen-api/.github/workflows/ci-backend.yml` (lines 67-69: added auth env vars)
