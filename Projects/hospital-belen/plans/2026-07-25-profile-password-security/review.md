VERDICT: SHIP

## Razonamiento
- Endpoint `PUT /users/me/password` correcto y seguro: valida current pw (constant-time vía scheme), salt `user.id` consistente con login (application/user/mod.rs:136), `new_password` min 8 (DTO), invalida session + permissions cache, retorna 200 con message. ✅
- Profile UI: initials avatar con tint por color de rol, email, badge, memberSince — todo desde datos reales (`/users/me` + `/users/me/roles`), sin DUMMY/María/setTimeout-mock residual. ✅
- Password flow: validación cliente (min 8 + confirm match) → submit → toast + redirect a /login tras 1.5s. Redirect correcto (backend invalida la sesión). ✅
- i18n: los 12 keys `profile.security.*` existen en es.json y todos los referenciados resuelven. ✅

## Hallazgos
🟡 MEDIO: `ProfilePage.vue:78` placeholder hardcodeado `"≥ 8 caracteres"` mientras los campos vecinos (74, 82) usan `t('profile.security.*')`. Contradice el claim "no hardcoded text" del dispatch. Ya existe key `profile.security.newPassword`; usar ese o agregar un `newPasswordHint`.
🟢 BAJO: `ProfilePage.vue:176` detección de "current pw incorrecta" por string-match (`includes('credential') || includes('401')`) sobre el mensaje de error. Frágil ante cambios de wording/formato del error. Preferible chequear el status HTTP (401) explícito del response. No bloquea.
🟢 BAJO: `get_me_permissions` sigue haciendo N+1 (heredado del cambio previo) — OK para 1-3 roles/usuario.

## Nota de scope
- Cambios previos (role colors, profile real-data) siguen en el workspace sin commit — el diff acumula todo. Los .DS_Store untracked del submódulo api no deben commitearse.
