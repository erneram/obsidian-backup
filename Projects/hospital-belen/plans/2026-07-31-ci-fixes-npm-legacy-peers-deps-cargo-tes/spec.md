# Spec — CI fixes (D1 + D2, defer D3)

Scope: fix hospital-belen CI. **D3 (frontend test framework) is explicitly deferred — do not touch.**
Infra-only; no product code. Both files live in submodule repos — Coder commits to the branch in
`.pipeline/branch.md` (reused from a prior run; leave the branch name as-is). Keep D1 and D2 in
separate commits.

---

## D1 — Frontend CI install flag

**File:** `hospital-belen-web/.github/workflows/ci-frontend.yml`

The 3 jobs (`typecheck`, `lint`, `build`) run `npm ci`; `deploy.yml` uses
`npm ci --legacy-peer-deps`. A peer conflict exists (that's why deploy carries the flag), so the CI
jobs fail at install today. Add `--legacy-peer-deps` to all three `npm ci` lines (27, 51, 79):

```yaml
run: npm ci --legacy-peer-deps
```

Nothing else changes.

**Verify:** CI-frontend goes green on the PR (install no longer errors).

---

## D2 — Backend `cargo test` job

**File:** `hospital-belen-api/.github/workflows/ci-backend.yml`

The `tests/` suites are **end-to-end**: `tests/common/mod.rs` does an HTTP `login()` against a
running server (`api_url()` → `http://localhost:8080`), not direct DB access. The server
auto-runs migrations + seeds the Belén tenant on boot
(`src/infrastructure/database/migrator.rs`; main.rs boot). So the job must boot Postgres **and** the
server before `cargo test`.

Add a `test` job (`needs: check`, parallel to `build`):

```yaml
test:
  name: Test
  runs-on: ubuntu-latest
  needs: check
  services:
    postgres:
      image: postgres:18-alpine
      env:
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: postgres
        POSTGRES_DB: hospital_belen_test
      ports:
        - 5432:5432
      options: >-
        --health-cmd "pg_isready -U postgres -d hospital_belen_test"
        --health-interval 5s --health-timeout 5s --health-retries 5
  env:
    APP_ENV: development                 # dev mode → APP_ENCRYPTION_KEY not required (src/config.rs:94-110)
    SERVICE_DB_URL: postgres://postgres:postgres@localhost:5432/hospital_belen_test
    TEST_DATABASE_URL: postgres://postgres:postgres@localhost:5432/hospital_belen_test
    TEST_API_URL: http://localhost:8080
    APP_PORT: "8080"
  steps:
    - uses: actions/checkout@v5
    - uses: dtolnay/rust-toolchain@stable
    - uses: actions/cache@v5
      with:
        path: |
          ~/.cargo/registry
          ~/.cargo/git
          target
        key: ${{ runner.os }}-cargo-test-${{ hashFiles('**/Cargo.lock') }}
        restore-keys: ${{ runner.os }}-cargo-test-
    - name: Start API server (auto-migrates + seeds on boot)
      run: cargo run --release &
    - name: Wait for server health
      run: |
        for i in $(seq 1 60); do
          curl -sf http://localhost:8080/ >/dev/null && exit 0
          sleep 5
        done
        echo "server did not become healthy" && exit 1
    - name: cargo test
      run: cargo test --all-targets
```

### Notes / edge cases (verify while implementing — effort=low, so keep minimal)
- **Health endpoint:** confirm the real health/ready path (`GET /` vs `/health` vs `/api/health`).
  Check `src/web/router.rs`; use whatever returns 200 unauthenticated. Adjust the wait loop URL.
- **Required boot env:** `src/config.rs` — in `APP_ENV=development` the encryption key defaults to
  zero (no secret needed) and `cookie_secure=false`. If boot still panics on a missing var, add it
  (candidates: `APP_DEFAULT_TENANT_ID` = `10000000-0000-0000-0000-000000000001` to match the seeded
  tenant). Add only what the panic demands — don't front-load optional SMTP/Twilio vars (all `.ok()`).
- **DB var name:** server reads `SERVICE_DB_URL` (see `.env.example`); tests read
  `TEST_DATABASE_URL`/`DATABASE_URL` (`tests/common/mod.rs:11-14`). Both point at the same CI DB.
- **Seed migration 003** must apply cleanly on an empty DB (it's idempotent) — tests depend on the
  Belén seed data existing.
- **Don't gate `build` on `test`** — keep them independent so a flaky server boot doesn't block the
  release build.

---

## D3 — DEFERRED
No frontend test framework/tests exist; standing up Vitest is a separate initiative. **Out of scope
for this run — do not add any frontend test job or `test` script.**

SKILL_RECOMENDADA: none applicable.
