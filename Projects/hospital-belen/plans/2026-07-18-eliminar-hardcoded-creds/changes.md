# Changes — stage-2: eliminar credenciales demo hardcodeadas

## `src/modules/auth/pages/LoginPage.vue`
- Eliminado bloque "Credenciales de prueba" (líneas 45-100 originales): 9 cuentas clickeables con passwords `111111`–`555555`.
- Eliminado helper `fill(email, password)` (era exclusivo del bloque demo).

## `src/modules/platform/pages/PlatformLoginPage.vue`
- Eliminado bloque `<!-- Dev credentials box -->`: `superadmin@inksight.io` / `111111`.
- Eliminado helper `fill(email, password)`.

## Verificación post-fix
```
grep -rniE "111111|222222|333333|444444|555555|hospitalbelen|@lapaz|@inksight\.io|@system\.local" src/
```
→ 0 resultados de credenciales. Solo falsos positivos: `@inksightdev/ui` (nombre de paquete).

## Tester review points
1. Login page renderiza sin ningún bloque de credenciales de prueba.
2. Platform login page ídem.
3. El grep anterior devuelve vacío (excluyendo `@inksightdev/ui`).
4. Flujo de login normal (email + password manual) funciona sin errores de compilación.
