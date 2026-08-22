# Changes — CI improvements + Node 24 update

## Web (`fix/ci-repairs`)

### Commit: style: prettier format kairosaid/ (one-time repo-wide reformat)
- `kairosaid/src/**` + `kairosaid/*.{js,ts,mjs,json}` — 127 files reformatted
- Fixes "Format check (Prettier)" CI step that was failing across all files
- `// ponytail: one-time repo-wide format, gate keeps it clean after`

### Commit: ci: split build job, node 22→24, bump checkout/setup-node/aws-creds to v5
- `.github/workflows/ci-frontend.yml`:
  - Renamed `typecheck-and-build` → `typecheck` (vue-tsc only)
  - Added separate `build` job (`needs: [typecheck, lint]`, runs `vite build`)
  - `node-version: '22'` → `'24'` (aligns with Dockerfile already on node:24-alpine)
  - `actions/checkout@v4` → `v5` (node24 action runtime)
  - `actions/setup-node@v4` → `v5` (node24 action runtime)
- `.github/workflows/deploy.yml`:
  - `node-version: 22` → `24`
  - `actions/checkout@v4` → `v5`
  - `actions/setup-node@v4` → `v5`
  - `aws-actions/configure-aws-credentials@v4` → `v5`
  - `deploy.yml` has `workflow_dispatch` trigger — human should run it once on branch to verify S3/CloudFront before merging to main

## API (`merge/platform-admin-to-main`)

### Commit: fix(ci): allow dead_code on db_url test helper
- `tests/common/mod.rs`: added `#[allow(dead_code)]` on `db_url()` — test helper visible to some targets but unused by `tenant_isolation` test binary

### Commit: ci: bump checkout@v4→v5 (node24 action runtime)
- `.github/workflows/ci-backend.yml`: `actions/checkout@v4` → `v5` in both `check` and `build` jobs
- `actions/cache@v4` left as-is (spec: verify only; no v5 exists yet)

---

## What Tester should verify

### Web
- `npx prettier --check .` → clean (no files flagged)
- `./node_modules/.bin/vue-tsc -b --noEmit` → exit 0
- CI shows 3 jobs: `Type Check`, `Lint & Format`, `Build` (Build waits for both)
- No `node20` deprecation warnings in CI logs
- `deploy.yml` `workflow_dispatch` run succeeds (S3 sync + CloudFront invalidation)

### API
- `cargo check --all-targets` with `RUSTFLAGS: -D warnings` → exit 0
- No dead_code warnings in CI logs
- No `node20` deprecation warnings in `checkout` step logs
