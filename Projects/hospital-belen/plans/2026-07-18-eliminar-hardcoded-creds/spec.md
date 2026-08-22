# Spec — Auditoría de seguridad: credenciales hardcodeadas (frontend)

Auditoría stage-1 effort=high sobre `hospital-belen-web/kairosaid`. Scan ya ejecutado.
Este spec = hallazgos + dónde seguir buscando + recomendaciones de fix.

---

## HALLAZGO CRÍTICO — Credenciales demo hardcodeadas, NO gated, en bundle de producción

Bloques de "Credenciales de prueba" con emails + passwords reales, wrapped solo en
`<template v-else>` / `<div>` — **sin `import.meta.env.DEV`**. El comentario dice
"solo desarrollo" / "Dev credentials box" pero renderizan en prod y están en `dist/`.

**`src/modules/auth/pages/LoginPage.vue` (líneas 45-99)** — 9 cuentas clickeables:
- `superadmin@system.local` / `111111`
- `admin@hospitalbelen.com` / `222222`, `doctor@…` / `333333`, `nurse@…` / `444444`, `secretary@…` / `555555`
- tenant lapaz: `admin@lapaz.com`/`222222`, `doctor@lapaz.com`/`333333`, `nurse@…`/`444444`, `secretary@…`/`555555`
- Helper `fill(email, password)` en línea 156.

**`src/modules/platform/pages/PlatformLoginPage.vue` (líneas 9-15)**:
- `superadmin@inksight.io` / `111111`. Helper `fill()` línea 77.

Riesgo: cualquiera que abra el login en prod ve credenciales de superadmin válidas
(si el seed corrió en ese entorno). Passwords secuenciales `111111`…`555555` = seed débil.

---

## HALLAZGO BAJO — PII en localStorage

`src/services/auth.service.ts:71,89` — guarda el objeto `user` completo en `localStorage`
(`setItem('user', JSON.stringify(user))`, `getItem('user')`). Legible por cualquier XSS.
**No es credencial**: el token de auth es cookie **HttpOnly** (`api.service.ts:16`
`withCredentials: true // required for HttpOnly cookie-based auth`). El token NO está expuesto. ✅

---

## LIMPIO (verificado, sin hallazgos)
- **No** hay `password/secret/apiKey/token = "literal"` en código fuente (grep exhaustivo).
- Refs de password en forms (`TenantAccordion`, `UserListPage`, `ProfilePage`,
  `AdminTenantSection`) inicializan en `ref('')` — correcto, sin prefill.
- **Ningún `.env` trackeado en git** (solo `.env.example`).
- **No** hay `import.meta.env.*` con nombres `pass/secret/key/token`.
- **No** hay credenciales en tests/mocks (`*.spec.ts`, `*mock*`).
- Login forms default `{ email: '', password: '' }` — sin prefill salvo los botones demo.

---

## Dónde buscar (checklist reproducible para reviewer)

Ejecutar desde `hospital-belen-web/kairosaid/src` (excluir SIEMPRE `dist/`, `node_modules/`):

1. **Asignaciones de credencial**:
   `grep -rniE "(password|passwd|pwd|secret|apikey|api_key|token|credential)\s*[:=]\s*['\"][^'\"]{4,}['\"]"`
2. **Refs/estado prellenado**: `grep -rniE "ref\(['\"][^'\"]{4,}['\"]\).*(pass|secret|token)"`
3. **Botones/helpers de autocompletar**: `grep -rniE "fill\(['\"]"` — buscar demo creds.
4. **Gating dev**: cerca de cada bloque de creds, verificar `v-if="import.meta.env.DEV"`.
5. **Storage**: `grep -rniE "(local|session)Storage\.(set|get)Item"` — qué se persiste.
6. **Env en código**: `grep -rniE "import\.meta\.env\.[A-Z_]+"` filtrando pass/secret/key/token.
7. **Comentarios sospechosos**: `grep -rniE "//.*(password|contraseña|credential|hardcod|demo user|TODO.*auth)"`.
8. **Tests/mocks/seeds**: mismos patrones en `*.spec.ts`, `*.test.ts`, `*mock*`, `*seed*`.
9. **Git**: `git ls-files | grep -iE "\.env"` — ningún `.env` real committeado.
10. **Bundle**: confirmar que tras el fix, `grep -r "111111" dist/` no devuelve nada.

---

## Recomendaciones de fix (para el coder — NO es código, es dirección)

Decisión del humano: **el seed NO corre en producción** → no hay rotación de cuentas
pendiente. El fix es puramente frontend: eliminar los bloques demo.

### 1. ELIMINAR los bloques demo (CRÍTICO) — acción definitiva, no gatear

**`LoginPage.vue`**: borrar el bloque completo de "Credenciales de prueba"
(`<div class="mb-6 ...">` línea 46 hasta su cierre ~línea 99, incluyendo el
`<p>Haz clic en cualquier credencial…</p>`). Borrar también el helper `fill()`
(líneas 156-160) — queda sin uso tras quitar los botones.

**`PlatformLoginPage.vue`**: borrar el `<!-- Dev credentials box -->` (`<div>` línea 10
hasta su cierre) y el helper `fill()` (línea 77).

No conservar los emails/passwords en ningún lado del repo. Documentar credenciales de
dev local, si hacen falta, en un `.env.local`/README NO committeado.

Verificar post-fix (debe devolver vacío):
```
grep -rniE "111111|222222|333333|444444|555555|@hospitalbelen|@lapaz|@inksight|@system.local" src dist
```

### 2. localStorage `user` → sesión HttpOnly (BAJO)
El token ya es cookie HttpOnly; el objeto `user` sigue en `localStorage`
(`auth.service.ts:71,89`) y es legible por XSS.

Recomendación (baja prioridad): **no persistir `user` en storage JS-legible**.
Mantenerlo solo en memoria (store Pinia) y rehidratar en cada carga vía `/users/me`
(que ya usa la cookie HttpOnly). Así toda la identidad de sesión vive detrás de la
cookie HttpOnly y nada sensible queda en localStorage.
- Cambios: `getMe()` deja de hacer `setItem('user', …)`; `getCurrentUser()`/
  `isAuthenticated()` leen del store en memoria, no de localStorage.
- Trade-off: un refresh dispara `/users/me` (1 request extra) en vez de leer localStorage.
  Aceptable. No urgente; puede ir en un fix aparte.

---

## Archivos afectados
- `src/modules/auth/pages/LoginPage.vue` — eliminar bloque demo + helper `fill()`.
- `src/modules/platform/pages/PlatformLoginPage.vue` — eliminar bloque demo + helper `fill()`.
- `src/services/auth.service.ts` — (opcional, bajo) `user` fuera de localStorage → memoria + `/users/me`.

## SKILL_RECOMENDADA
`security-review` (built-in) para el diff del fix antes del sign-off.

## Notas
- Effort high: scan reproducible incluido (sección "Dónde buscar").
- Seed NO corre en prod (decisión humana) → sin rotación de cuentas. Fix = solo borrar bloques.
- El único vector real de fuga son los botones demo. Todo lo demás está limpio.
- Prioridad: #1 (eliminar bloques) inmediato; #2 (localStorage → memoria) opcional, fix aparte.
