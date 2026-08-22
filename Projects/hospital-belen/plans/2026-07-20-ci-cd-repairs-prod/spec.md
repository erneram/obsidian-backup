# Spec — Auditoría CI/CD (web + api)

Stage-1 effort=high. App EN PRODUCCIÓN → cambios cuidadosos. Runs reales inspeccionados con `gh`.

## Inventario de workflows

**Web** (`hospital-belen-web/.github/workflows/`):
| Archivo | Trigger | Qué hace | Estado real |
|---|---|---|---|
| `ci-frontend.yml` | push/PR main | typecheck+build (job A) · lint+format (job B) | 🔴 FAIL |
| `deploy.yml` | push main + manual | build → S3 sync + CloudFront invalidate (`kairosaid.inksightdev.com`) | ✅ OK — **prod real** |
| `test-pipeline.yml` | push main | Docker build+push a ghcr `frontend-test-*` | 🔴 FAIL |
| `prod-pipeline.yml` | release published | Docker build+push ghcr `frontend-prod-*` + SSH deploy a `128.24.93.178` | ⏸️ solo en release |

**API** (`hospital-belen-api/.github/workflows/`):
| Archivo | Trigger | Qué hace | Estado real |
|---|---|---|---|
| `ci-backend.yml` | push/PR main | check+clippy+fmt (job A) · build release (job B) | 🔴 FAIL |
| *(sin archivo)* "Deploy API to App Runner" | ? | aparece en runs `[ok]` pero **no hay .yml en el repo** | ⚠️ orphan/managed fuera |

---

## POR QUÉ FALLAN (causa raíz, confirmada en logs)

### F1 — Frontend: `package-lock.json` desincronizado (CRÍTICO, bloquea todo)
Log: `npm error ... npm ci can only install when package.json and package-lock.json are in sync.
Missing: hono@4.12.31 from lock file`.
- Rompe **ambos jobs de `ci-frontend.yml`** (mueren en `npm ci` antes de typecheck/lint) y
  el Docker build de `test-pipeline.yml` (que corre `npm ci` dentro del Dockerfile).
- Dato: `hono` **no está en `package.json`** y **nadie lo importa** (grep vacío), pero está
  en el lockfile con versión distinta a la que resuelve el árbol → lockfile stale/sucio.
- **Por qué `deploy.yml` SÍ pasa**: usa `npm ci --legacy-peer-deps` (L31), que relaja la
  validación y "se salva". Frágil — depende de un flag, no de un lockfile sano.
- **Fix**: `cd kairosaid && npm install` → regenera `package-lock.json` limpio → commit.
  Verificar que `hono` desaparece del lock (o queda con la versión correcta si es transitivo).
  `npm ci` (sin `--legacy-peer-deps`) debe pasar localmente antes de commitear.

### F2 — Frontend: script `format` mal configurado (rompe job lint, aflora tras F1)
- `package.json`: `"format": "prettier --write src/"` (**escribe**).
- `ci-frontend.yml:64`: `npm run format -- --check` → ejecuta `prettier --write src/ --check`
  → **`--write` y `--check` son mutuamente excluyentes** → prettier sale con error.
- Además, aunque no fallara, un check en CI NUNCA debe usar `--write` (mutaría archivos).
- **Fix**: añadir script `"format:check": "prettier --check src/"` y en CI L63-64 llamar
  `npm run format:check`. Dejar `format` (write) para uso local.

### F3 — Backend: `-D warnings` promueve warning a error (CRÍTICO)
- Log: `error: unused import: web::error::WebError` → `cargo check` sale con 101.
- `ci-backend.yml`: `RUSTFLAGS: -D warnings` (L11) + `cargo clippy -- -D warnings` (L41).
  Cualquier warning (imports sin usar, etc.) tumba el CI. **El CI hace bien su trabajo**;
  el problema es código con warnings.
- **Fix (código, no CI)**: `cargo fix` / limpiar imports sin usar. Mantener `-D warnings`
  como quality gate — es correcto para un backend en prod. Es tarea del coder.
- Alternativa NO recomendada: relajar a warnings no-fatales (perdería el gate).

---

## Redundantes / no-usados — DECISIÓN TOMADA

Prod web = `deploy.yml` (S3/CloudFront). Prod API = App Runner. Los demás caminos se remueven.

### R1 — `test-pipeline.yml` → **ELIMINAR** (confirmado)
- `hospital-belen-web/.github/workflows/test-pipeline.yml`.
- Build+push Docker `frontend-test-*` a ghcr, mismo trigger `push main` que `deploy.yml`,
  **sin deploy**. Nadie consume la imagen; siempre falla; quema minutos. `git rm` el archivo.

### R2 — `prod-pipeline.yml` → **ELIMINAR** (confirmado — infra muerta)
- `hospital-belen-web/.github/workflows/prod-pipeline.yml`.
- SSH a `128.24.93.178` NO es prod (prod es S3/CloudFront). Workflow muerto → `git rm`.
- **Acción de seguridad obligatoria al remover**: ver S1 (rotar `SSH_PRIVATE_KEY_PROD`).

### R3 — Deploy API (App Runner) → **MANTENER, no hay nada que tocar**
- Confirmado: **no existe `.yml` de deploy en `hospital-belen-api/`** (solo `ci-backend.yml`).
  El deploy a App Runner es **gestionado fuera del repo** (App Runner auto-deploy desde
  ECR/source connection), no un GitHub workflow.
- **No crear** un workflow de deploy API — App Runner ya lo maneja. Dejar como está.
- (Opcional) documentar en el README del api que el deploy es App Runner managed, para que
  nadie reintroduzca un workflow que compita.

---

## Qué es CRÍTICO para prod (NO romper)
- `deploy.yml` (web → S3/CloudFront) — **camino de deploy real, funciona**. No tocar salvo
  hardening S2. Migrar su `npm ci --legacy-peer-deps` a `npm ci` limpio DESPUÉS de F1.
- Deploy backend (App Runner) — funciona. Verificar/documentar (R3), no romper.
- `ci-frontend.yml` + `ci-backend.yml` como quality gates — **arreglar y conservar**, no eliminar.

---

## Seguridad (app en prod)

### S1 — `prod-pipeline.yml`: SSH muerto → limpiar credencial (confirmado)
- Server `128.24.93.178` ya NO es prod → al remover el workflow (R2):
  - **Rotar/revocar `SSH_PRIVATE_KEY_PROD`** en el server viejo (por si el key sigue vivo).
  - **Borrar el secret `SSH_PRIVATE_KEY_PROD`** de GitHub → Settings → Secrets (queda huérfano).
  - Revisar si `VITE_API_URL_PROD`, `VITE_API_HOST_PREFIX_PROD` (usados solo por prod-pipeline)
    quedan huérfanos → borrar si nada más los usa. (`deploy.yml` usa su propio `VITE_API_URL` inline.)
- Nota: rotación de key / borrado de secrets = **acción humana en GitHub/servidor**, no del coder.

### S2 — `deploy.yml`: credenciales AWS estáticas (hardening, no urgente)
- Usa `secrets.AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` (llaves long-lived).
- **Recomendación**: migrar a OIDC (`aws-actions/configure-aws-credentials` con
  `role-to-assume`) → credenciales efímeras, sin secretos long-lived en GitHub.
  Cambio contenido, alto valor de seguridad. Post-estabilización.

### S3 — Menores
- `provenance: false` en los build-push Docker → sin attestation de supply-chain. Si se
  conserva algún pipeline Docker, considerar `provenance: true`.
- `GITHUB_TOKEN` con `packages: write` está bien acotado. OK.

---

## Plan de ejecución (todo cero-riesgo prod — solo CI/lockfile/código + borrar workflows muertos)

1. **F1** — regen `package-lock.json` (`cd kairosaid && npm install`), verificar que `hono`
   sale del lock, `npm ci` limpio pasa local → commit. Desbloquea `ci-frontend` (ambos jobs).
2. **F2** — añadir script `"format:check": "prettier --check src/"`, cambiar `ci-frontend.yml:63-64`
   a `npm run format:check`.
3. **F3** — limpiar warnings del backend (unused import `WebError` y otros) → `cargo check`/
   `clippy` en verde. CI intacto.
4. **R1** — `git rm hospital-belen-web/.github/workflows/test-pipeline.yml`.
5. **R2** — `git rm hospital-belen-web/.github/workflows/prod-pipeline.yml`.
   → luego S1 (humano: rotar SSH key + borrar secrets huérfanos).
6. **Mantener** intactos: `deploy.yml` (web S3), `ci-frontend.yml`, `ci-backend.yml`, deploy
   API App Runner (managed fuera del repo).
7. **S2** — (opcional, ventana aparte) migrar `deploy.yml` a OIDC. No bloquea el cleanup.

Todo el diff del coder es: 1 lockfile, 1 script package.json, 1 edit ci-frontend.yml,
N imports Rust, y 2 workflows borrados. Ningún path de deploy se modifica.

## Verificación (tester)
- Tras F1: en `kairosaid/`, `rm -rf node_modules && npm ci` (sin `--legacy-peer-deps`) pasa.
- Tras F2: `npm run format:check` corre sin el error de flags.
- Tras F3: `cargo check --all-targets` y `cargo clippy --all-targets -- -D warnings` en verde.
- Confirmar en Actions que `CI Frontend` y `CI Backend` quedan verdes en el PR.
- **NO** disparar `deploy.yml` ni deploy backend como parte del test — son prod.

## OPEN QUESTIONS — RESUELTAS (decisión humana)
1. ✅ Prod web = `deploy.yml` (S3/CloudFront). `prod-pipeline.yml` (SSH) = muerto → eliminar.
2. ✅ `test-pipeline.yml` = eliminar (no se cablea staging).
3. ✅ Deploy API = App Runner (managed fuera del repo, sin .yml). Mantener, no reintroducir.

Pendiente solo para el humano (no coder): rotar `SSH_PRIVATE_KEY_PROD` + borrar secrets huérfanos (S1).

## SKILL_RECOMENDADA
`security-review` sobre el diff antes del sign-off (toca workflows + secretos).

## Notas
- Effort high: 5 workflows leídos, logs de fallo reales por repo, causa raíz confirmada.
- Las fallas son 3 causas concretas (F1 lockfile, F2 prettier flags, F3 warnings), no un
  problema difuso. F1 arregla la mayoría de los checks rojos de un solo commit.
- Cero riesgo para prod en pasos 1-3 (solo tocan CI/lockfile/código, no los deploy).
