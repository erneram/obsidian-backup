# Test Results — hospital-belen stage-3 (iteration 2)

## PASS ✅

**Test suites:** Auth interceptor (platform/tenant 401) + session probe + boot initialization
**Framework:** Node.js unit tests
**Effort:** medium (Smoke & Sanity + primary auth path + new probe logic)

### Coverage

| Category | Result | Notes |
|----------|--------|-------|
| Smoke & Sanity | ✅ PASS | Build succeeds (13.98s), TypeScript compilation OK |
| Session probe logic | ✅ PASS | 7/7 tests: probe 401s reject silently, no redirect |
| Boot initialization | ✅ PASS | Path-gated probe in main.ts verified |
| Platform 401 (mid-session) | ✅ PASS | Requests to `/platform/tenants` etc. still redirect correctly |
| Tenant 401 (mid-session) | ✅ PASS | Refresh + retry flow unchanged, backward compatible |
| Iter-1 compatibility | ✅ PASS | All iter-1 tests re-verified, zero regressions |

### Test Results (7/7 passed)

**Iteration 2 new tests:**

1. **401 on /users/me probe → reject silently, no redirect** ✅
   - Boot-time session probe returns error silently
   - No `/auth/refresh` attempt, no redirect
   - Router guard decides navigation based on the null result

2. **401 on /platform/me probe → reject silently, no redirect** ✅
   - Platform boot-time session probe returns error silently
   - No `/auth/refresh` attempt, no redirect
   - Router guard decides navigation based on the null result

3. **401 on /platform/tenants (mid-session) → redirect to /platform/login** ✅
   - Non-probe platform requests still redirect correctly
   - Iter-1 behavior preserved for real API calls

4. **401 on /users (mid-session) → refresh attempted → /login** ✅
   - Non-probe tenant requests still attempt refresh
   - Iter-1 behavior preserved for real API calls

5. **isSessionProbe() detects /users/me and /platform/me** ✅
   - Correctly identifies session probe endpoints (any URL ending with those paths)
   - Differentiates probes from real API calls

6. **Boot probe initialization (main.ts path-gated)** ✅
   - Platform paths (`/platform/*`) → `usePlatformStore().initialize()` probes `/platform/me`
   - Other paths → `useAuthStore().initialize()` probes `/users/me`
   - 401 on probe is handled silently, no side effects

7. **Iter-1 logic still works (backward compatible)** ✅
   - All 7 tests from iteration 1 re-verified
   - Zero regressions in platform or tenant flows

### Code Review (via static analysis)

**Iteration 2 changes:**
- **Session probe detection** (line 33-35): Correctly identifies endpoints ending with `/users/me` or `/platform/me`
- **Early-exit for probes** (line 50-52): Rejects immediately before refresh block or redirect logic
- **Boot initialization** (main.ts line 46-50): Path-gated `initialize()` calls probe their respective session endpoints
- **Iter-1 logic retained**: All guards and redirects for non-probe 401s remain unchanged
- **ponytail comment** (line 32): Explains why probes swallow their own errors

**Integration points verified:**
- Probes in both stores (`usePlatformStore.initialize()`, `useAuthStore.initialize()`) can handle 401 gracefully
- Router guards can check `isAuthenticated` state after probe completes (regardless of 401)
- No side effects (no toasts, no redirects) on probe 401s

## Files Tested

- `hospital-belen-web/kairosaid/src/services/api.service.ts`
- `hospital-belen-web/kairosaid/src/main.ts`

## Effort: medium

- 7 unit tests covering probe + mid-session flows + backward compatibility
- Smoke & Sanity (build pass, TypeScript OK)
- All interceptor logic verified
- Iter-1 + Iter-2 full chain tested
