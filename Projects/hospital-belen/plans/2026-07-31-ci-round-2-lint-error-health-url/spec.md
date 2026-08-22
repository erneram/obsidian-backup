# Spec — CI fixes round 2 (diagnosed from actual failing runs)

Previous CI changes landed but both pipelines are still red. Root causes below are from the real
GitHub Actions logs, not guesses. Infra/lint only — no product logic. Coder commits to the branch in
`.pipeline/branch.md` (reused). D3 still deferred — do not add frontend tests.

---

## F1 — Frontend CI: ESLint error (1 line)

**Failing run:** `Lint & Format` job, exit 1. Type Check passes; Build is skipped because it
`needs: [typecheck, lint]`. Single blocking error (523 warnings are non-fatal — no `--max-warnings`):
```
201:27  error  Attribute ':model-value' can't be hyphenated   vue/attribute-hyphenation
```

**File:** `hospital-belen-web/kairosaid/src/modules/platform/pages/PlatformTenantDetailPage.vue:201`
```html
<!-- current -->
<Progress :model-value="((5 - countdown) / 5) * 100" class="h-2 w-full" />
<!-- fix: camelCase the bound prop -->
<Progress :modelValue="((5 - countdown) / 5) * 100" class="h-2 w-full" />
```
Only the **bound prop** `:model-value` is illegal. The `@update:model-value` **event** listeners
elsewhere are fine (the rule targets attributes, not events) — do not touch them.

**Note on D1:** the earlier `npm ci → npm ci --legacy-peer-deps` change is **moot** — `npm ci`
installs cleanly in CI (Type Check job got past install). Don't bother adding the flag; if it was
already added, harmless. The peer-dep hypothesis was wrong; the real blocker is this lint error.

**Verify:** `cd kairosaid && npm run lint` exits 0.

---

## F2 — Backend CI: `Test` job — "server did not become healthy"

**Failing run:** `check` and `build` pass; **`Test`** fails. The health-wait loop ran the full 300s
then exited 1. Diagnosis from the log + code:

1. **Wrong health URL (primary).** The probe hits `http://localhost:8080/` with `curl -sf`, but the
   only public health route is **`/health`** (`src/web/router.rs:72`, unauthenticated). `/` returns
   404 → `curl -sf` fails every iteration → the loop can never pass even when the server is up.
2. **Release compile runs inside the health window.** `cargo run --release &` backgrounds a *cold
   release build*; a large axum API can take >5 min to compile, so the server isn't even listening
   within the 300s budget. Build must happen in a foreground step (cached), then run the binary.
3. **No server logs.** The backgrounded server's output is discarded, so failures are invisible.

**Confirmed non-issues (do not add):**
- Redis: `RedisCache` is an in-memory stub (`redis_cache.rs`) — `from_env()` never connects and
  `ping()` returns `true`. No Redis service or `REDIS_URL` needed. `/health` returns 200 on DB alone.
- Test users: tests log in as `admin@inksight.com`, `superadmin@system.local`,
  `dr.mendoza@hospitalbelen.gt`, etc. These come from `dev_seed::run`, which runs on boot when
  `APP_ENV != production` (`src/infrastructure/dev_seed.rs`). Our job sets `APP_ENV=development`, and
  the server only starts listening *after* boot seeding completes, so the users exist before tests
  run. Keep `APP_ENV=development`.
- Encryption key: dev mode defaults it to zero — no `APP_ENCRYPTION_KEY` needed.

**File:** `hospital-belen-api/.github/workflows/ci-backend.yml` — replace the `test` job's
server-start + health-wait + test steps (keep the `services.postgres` + `env` block; the binary is
named `app`). Use a **debug** build — far faster to compile and it shares the target dir with
`cargo test`:

```yaml
    - name: Build server + tests (debug, cached)
      run: cargo build --all-targets
    - name: Start API server (auto-migrates + dev-seeds on boot)
      run: ./target/debug/app > server.log 2>&1 &
    - name: Wait for server health
      run: |
        for i in $(seq 1 60); do
          if curl -sf http://localhost:8080/health >/dev/null; then
            echo "server healthy after ${i} tries" && exit 0
          fi
          sleep 2
        done
        echo "::error::server did not become healthy — dumping server.log"
        cat server.log
        exit 1
    - name: cargo test
      run: cargo test --all-targets
```

### Edge cases / verify while implementing
- **Health returns 200 only when the DB pool is up** (`health.rs` does `SELECT 1`); the server binds
  only after migrations + dev_seed finish, so a 200 means the DB is ready — no test/seed race.
- **Migration 003** (2726 package_items) runs on boot; it's idempotent but adds boot time — the
  60×2s=120s window after a completed build is enough, but if boot is slow bump retries, don't
  shorten. Keep the build **out** of the wait window (that's the whole point of F2.2).
- **Port/host:** config defaults `APP_HOST=0.0.0.0`, `APP_PORT=8080` — `localhost:8080` reachable.
  The job already exports `APP_PORT=8080`; keep it.
- **`cargo build --all-targets`** compiles the test binaries too, so the later `cargo test` is nearly
  instant and its recompile can't blow the health budget.
- If `curl` isn't preinstalled on the runner it is on `ubuntu-latest` — no action needed.

**Verify:** the `Test` job goes green; on any regression `server.log` is now in the run output.

---

## D3 — still DEFERRED (no frontend test job / framework).

SKILL_RECOMENDADA: none applicable.
