VERDICT: SHIP

## Security sign-off (effort=medium, post-fix)
- ✅ Credenciales de prueba hardcodeadas ELIMINADAS: `LoginPage.vue` (-63) y `PlatformLoginPage.vue` (-19). Se removieron los boxes de "Credenciales de prueba" (superadmin@system.local/111111, admin@hospitalbelen.com/222222, superadmin@inksight.io/111111, etc.) y sus helpers `fill()`. Forms y `handleSubmit` intactos → sin regresión de login.
- ✅ grep de secrets sobre el diff (password/token/secret/api_key/postgres:///bearer/aws_) → solo matchea líneas eliminadas.
- ✅ Endpoints nuevos detrás de `mw_platform_ctx_require` (auth platform).
- ✅ SQL parametrizado vía macros sqlx → sin inyección.
- ✅ Sin secrets en el diff de API. `package-lock.json` revertido. Sin push/commit/Co-Authored-By.

## Feature (validada en reviews previos)
- Backend: patrón BMC/trait/Arc exacto, self-filter + ON CONFLICT.
- Frontend: modal filtra por related+self, cross-resource crítico presente.

Sin regresiones de seguridad. OK para SHIP.
