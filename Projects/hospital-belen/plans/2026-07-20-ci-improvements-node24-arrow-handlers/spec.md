# Spec — CI improvements + Node 20 deprecation

Stage-1 effort=high. Current Actions state inspected live via `gh`. App in prod → careful.

## Current CI state (verified)

Cleanup from prior cycle **landed**: `test-pipeline.yml` + `prod-pipeline.yml` are gone.
Remaining workflows:
- Web: `ci-frontend.yml` (typecheck+build / lint+format), `deploy.yml` (S3/CloudFront, prod ✅).
- API: `ci-backend.yml` (check+clippy+fmt / build). Deploy = App Runner (managed outside repo).

A branch `fix/ci-repairs` is actively fixing CI. Lockfile (F1) is **already fixed** — `npm ci`
now passes. Both CIs are still red for the reasons below.

**Note**: CI was changed to run on all branches (not only main/master) — keep, it's useful.

---

## 2. What fails NOW (current errors, not last cycle's)

### Web — Prettier format check (only remaining failure)
- Install ✅, ESLint ✅. Fails at `ci-frontend.yml` step "Format check (Prettier)".
- Cause: `format:check` gate now works, but **the whole `src/` was never Prettier-formatted** —
  dozens of files flagged (`main.css`, most `.vue` under `components/`, etc.).
- **Fix**: one-time `cd kairosaid && npx prettier --write src/` → commit the reformat.
  Big but mechanical diff. `// ponytail: one-time repo-wide format, gate keeps it clean after`.
  - Do this in **its own commit** ("style: prettier format src/") so it doesn't bury logic diffs.
  - Coordinate: land it when no other big frontend branch is open (merge conflicts).

### API — `cargo check --all-targets` warning-as-error (only remaining failure)
- `RUSTFLAGS: -D warnings` + `--all-targets` (includes tests) → any warning fails.
- Current offender: `tests/common/mod.rs:7:8` (unused/dead code in a **test helper**).
  Prior offenders (`WebError` unused import, `db_url` dead_code) already being patched one by one.
- **Fix (code, coder)**: clean the warnings. For legit test-only helpers that look "unused"
  to some targets, prefer `#[allow(dead_code)]` on the specific item over silencing globally.
  Keep `-D warnings` — it's a correct prod gate.
- **Root-cause suggestion**: the whack-a-mole is because `--all-targets` lints tests too.
  Options: (a) keep and fix each (current path), or (b) run `cargo check` (lib/bins) strict and
  `cargo check --tests` non-fatal. **Recommend (a)** — tests deserve the gate too; just annotate
  the few intentional test helpers.

---

## 1. Frontend `build` verification step (mirror backend topology)

**Clarification**: `ci-frontend.yml` **already builds** — job `typecheck-and-build` runs
`npx vue-tsc -b --noEmit` then `npx vite build` (L31-37). So a build already runs in CI.

What backend does differently: **two jobs**, `check` (lint/typecheck) → `build` (`cargo build
--release`, `needs: check`). The ask = mirror that separation. Proposed restructure of
`ci-frontend.yml` into 3 jobs:

```yaml
jobs:
  typecheck:            # vue-tsc -b --noEmit   (rename from typecheck-and-build)
  lint:                 # eslint + prettier --check   (unchanged)
  build:
    needs: [typecheck, lint]
    steps: ... npx vite build   (env VITE_API_URL: http://localhost:8080)
```
- Move `vite build` out of the typecheck job into its own `build` job gated on `needs`.
- Benefit: build only runs if typecheck+lint pass (matches backend), clearer signal, and the
  build artifact step is isolated.
- Keep `VITE_API_URL: http://localhost:8080` for the CI build (verification only, not deployed).
- `// ponytail`: don't add artifact upload / caching-of-dist unless asked — YAGNI, it's a
  verification build.

---

## 3. Node 20 deprecation → Node 24 (action runtime, NOT the app's node)

**Critical distinction** (easy to get wrong): the changelog deprecates the **Node runtime that
JavaScript actions execute on** (node20 → node24), *not* the `node-version` your app builds with.
These are two separate changes — do **both**:

### 3a. App `node-version` 22 → 24 (decision taken)
- Currently `node-version: 22` in `ci-frontend.yml:22,51` and `deploy.yml:26`. Bump all to **24**.
- **Harmonization note**: `hospital-belen-web/Dockerfile` and `Dockerfile.dev` **already use
  `node:24-alpine`** — so CI/deploy on 22 was *inconsistent* with the Docker build. Moving CI to
  24 aligns them. Good change, not just churn.
- No `engines` field in `package.json`, no `.nvmrc` → nothing else pins node. Only the 3
  `node-version` inputs change. (Optional: add `"engines": { "node": ">=24" }` to lock intent.)
- Verify `npm ci` + `vite build` pass on node 24 in CI (Vite 5+/vue-tsc support 24 fine).

### 3b. Action runtime node20 → node24 (bump action versions)
The separate, real deprecation fix.

### Timeline (from changelog)
- **2026-06-16**: runners default to node24. **Fall 2026**: node20 removed entirely.
- Not an emergency today, but low-effort to fix now. Temporary escape hatch until fall 2026:
  env `ACTIONS_ALLOW_USE_UNSECURE_NODE_VERSION=true` (do NOT rely on this — fix versions instead).

### Action version bumps (3b)

| Action | Current | → | Where |
|---|---|---|---|
| `actions/checkout` | v4 | **v5** (node24) | ci-frontend, ci-backend, deploy |
| `actions/setup-node` | v4 | **v5** (node24) | ci-frontend, deploy |
| `aws-actions/configure-aws-credentials` | v4 | **v5** (node24) | deploy.yml |
| `actions/cache` | v4 | **verify** — pin latest v4 (recent v4.x ships node24); bump only if a node24 release exists | ci-backend |
| `dtolnay/rust-toolchain` | @stable | **no change** — Rust/composite action, not a node action | ci-backend |

- **VERIFY each bump's release notes** before applying — confirm the target major actually runs
  node24 and has no breaking input changes (checkout v5 / setup-node v5 are drop-in for our usage).
- `deploy.yml` touches **prod** → bump its actions on a branch, run the workflow via
  `workflow_dispatch` once to confirm S3 sync + CloudFront invalidation still work **before** main.

---

## Files to modify
- `hospital-belen-web/.github/workflows/ci-frontend.yml` — split `build` job (item 1) + `node-version` 22→24 (3a) + action bumps checkout/setup-node v5 (3b).
- `hospital-belen-web/.github/workflows/deploy.yml` — `node-version` 22→24 (3a) + action bumps checkout/setup-node/aws-creds v5 (3b). Verify via dispatch.
- `hospital-belen-api/.github/workflows/ci-backend.yml` — checkout v5, verify cache v4 runtime (no node-version there — Rust).
- `hospital-belen-web/kairosaid/` source — `prettier --write src/` reformat (own commit); optional `engines.node` in package.json.
- `hospital-belen-api/` source — clean remaining `-D warnings` offenders (coder, code not CI).

## Execution order (low risk → prod-adjacent last)
1. **Prettier reformat** `src/` (own commit) → web CI green on format.
2. **Backend warnings** cleanup → api CI green on check.
3. **Frontend `build` job split** → verify green.
4. **App node 22→24** (3a) in ci-frontend + deploy → verify `vite build` on 24.
5. **Action bumps** checkout/setup-node v5 in CI workflows + cache verify (3b).
6. **`deploy.yml`** node 24 + action bumps LAST → test via `workflow_dispatch` before merging to main.

## Verification (tester)
- Web: `npx prettier --check src/` clean; `npx vue-tsc -b --noEmit` clean; `npx vite build` ok;
  CI shows separate `typecheck` / `lint` / `build` jobs all green.
- API: `cargo check --all-targets` + `cargo clippy --all-targets -- -D warnings` green.
- Bumps: each workflow run shows no node20 deprecation warning in logs.
- `deploy.yml`: dispatch run succeeds (S3 sync + invalidation) — **do not** merge the deploy bump
  until that dispatch is green.

## OPEN QUESTIONS (human)
1. ✅ RESOLVED — app `node-version` bumped to **24** (aligns with Dockerfile already on node:24).
2. `deploy.yml` node bump + action bumps touch prod — OK to test via `workflow_dispatch` on a branch first?

## SKILL_RECOMENDADA
`security-review` on the diff (workflows + action versions = supply-chain surface).

## Notes
- effort high: live Actions state, current failing steps identified per repo, changelog read.
- The Node ask is mostly a no-op on `node-version` (already 22) — the substance is action bumps.
- Nothing here changes a deploy path's behavior; only `deploy.yml`'s action versions (verify-gated).
