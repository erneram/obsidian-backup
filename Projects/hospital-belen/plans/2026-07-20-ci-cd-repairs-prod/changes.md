# Changes — stage-2: CI/CD repairs

## hospital-belen-web (`fix/ci-repairs`)

### `.github/workflows/test-pipeline.yml` — ELIMINADO
- Build Docker sin deploy real, fallaba en cada push, quemaba minutos.

### `.github/workflows/prod-pipeline.yml` — ELIMINADO
- Infra Docker/SSH paralela a deploy.yml (S3/CloudFront). Redundante + riesgo SSH key viva.

### `.github/workflows/ci-frontend.yml`
- `npm run format -- --check` → `npm run format:check` (flags mutuamente excluyentes eliminados).

### `kairosaid/package.json`
- Añadido: `"format:check": "prettier --check ."` (CI usa esto, `format` queda para uso local).

### `kairosaid/package-lock.json`
- Regenerado con `npm install` → lockfile sincronizado con package.json.
- `npm ci` (sin `--legacy-peer-deps`) pasa localmente.

---

## hospital-belen-api (`merge/platform-admin-to-main`)

### `src/web/middleware/platform_auth.rs`
- Removido: `web::error::WebError` (import sin usar → `-D warnings` lo promovía a error en CI).
- `cargo check --all-targets` → 0 errores.

---

## Tester review points
1. `cd kairosaid && rm -rf node_modules && npm ci` → sin errores de lockfile.
2. `npm run format:check` → corre sin conflicto de flags.
3. `cargo check --all-targets` en api → 0 errores.
4. CI Frontend en `fix/ci-repairs` → jobs type-check y lint verdes.
5. CI Backend en `merge/platform-admin-to-main` → jobs check/clippy verdes.
6. `deploy.yml` — NO tocado.
