# Test Results — hospital-belen profile page: password change (medium effort)

**Status:** ✅ PASS  
**Effort:** medium  
**Date:** 2026-07-25  

---

## Coverage Summary

| Category | Tests | Result |
|---|---|---|
| Backend Validation | 3 | ✅ PASS |
| Frontend UI | 5 | ✅ PASS |
| Password Change Flow | 4 | ✅ PASS |
| Build & i18n | 2 | ✅ PASS |
| **Total** | **14** | **✅ PASS** |

---

## Backend Validation

**T1: PUT /api/users/me/password — current password verification**
- ✅ Lines 264–269: Fetch user auth-capable variant (includes password_hash)
- ✅ Lines 271–276: Validate current password via `validate_pwd()`, throws InvalidCredentials on mismatch
- ✅ Line 278: Change password only if validation passes

**T2: Session invalidation**
- ✅ Line 281: `state.cache.invalidate_session(uid)`
- ✅ Line 282: `state.cache.invalidate_user_permissions(uid)` (clears permission cache too)

**T3: Response**
- ✅ Line 284: Returns 200 with JSON message
- ✅ Message: "Password changed. Session invalidated."

---

## Frontend UI

**T4: Identity header (profile card)**
- ✅ Avatar: h-16 w-16 rounded-full with initials (lines 116–121)
  - Initials computed from name (first + last initial, or single initial)
- ✅ Avatar styling (lines 123–127):
  - Role color tinted background (`${c}26` = 15% opacity)
  - Solid role-color text
  - Fallback to muted bg/fg if no role color
- ✅ Display name: `profile.name` (real user name, no fake)
- ✅ Email: shown below name
- ✅ Role badge: colored via `roleBadgeStyle()` (line 129)
- ✅ Member-since: computed from `profile.memberSince` (lines 131–136), formatted as medium date (es-ES)

**T5: Seguridad section layout**
- ✅ ShieldCheck icon (18px, from lucide-vue-next, line 64)
- ✅ Title + subtitle with i18n (lines 66–67)
- ✅ 3-column grid on desktop, 1-column on mobile (grid-cols-1 sm:grid-cols-3, line 71)

**T6: Form fields (design-system)**
- ✅ Label + Input for current password (lines 73–74)
- ✅ Label + Input for new password (lines 77–78)
- ✅ Label + Input for confirm password (lines 81–82)
- ✅ All from @inksightdev/ui (`Label`, `Input`)
- ✅ Input types: `type="password"`

**T7: Error/success messages**
- ✅ Error display (line 86): red text, i18n'd
- ✅ Success display (line 87): green text, i18n'd
- ✅ Button state: disabled while saving (line 89)
- ✅ Button text: i18n'd (profile.security.save / saving)

**T8: Card component**
- ✅ `Card` + `CardContent` from @inksightdev/ui (line 61)
- ✅ Consistent with identity header card styling

---

## Password Change Flow

**T9: Client-side validation**
- ✅ Line 154–157: Minimum length 8 chars (i18n error: errorTooShort)
- ✅ Line 158–161: Passwords must match (i18n error: errorMismatch)

**T10: API call**
- ✅ Line 164: PUT `/users/me/password`
- ✅ Payload: `{ current_password, new_password }`
- ✅ Sends both fields to backend

**T11: Success flow**
- ✅ Line 170–173: On success:
  - `pwSuccess.value = t(...)` (shows in green)
  - `toast()` call with success title (line 172)
  - `router.push('/login')` after 1.5s (line 173)

**T12: Error handling**
- ✅ Line 176–177: Detect credential errors (credential/401) → errorWrongCurrent (i18n)
- ✅ Line 178–180: Generic errors → errorGeneric (i18n)
- ✅ Error messages shown in red (line 86)

---

## Build & i18n

**T13: Build passes**
- ✅ `npm run build` completes without TypeScript errors
- ✅ No new bundle size regressions

**T14: i18n strings (profile.security.* in es.json)**
- ✅ `profile.security.title`: "Seguridad"
- ✅ `profile.security.subtitle`: "Contraseña y configuración de acceso"
- ✅ `profile.security.currentPassword`: "Contraseña actual"
- ✅ `profile.security.newPassword`: "Nueva contraseña"
- ✅ `profile.security.confirmPassword`: "Confirmar contraseña"
- ✅ `profile.security.save`: "Actualizar contraseña"
- ✅ `profile.security.saving`: "Guardando..."
- ✅ `profile.security.success`: "Contraseña actualizada. Cerrando sesión..."
- ✅ `profile.security.errorMismatch`: "Las contraseñas no coinciden."
- ✅ `profile.security.errorTooShort`: "La contraseña debe tener al menos 8 caracteres."
- ✅ `profile.security.errorWrongCurrent`: "Contraseña actual incorrecta."
- ✅ `profile.security.errorGeneric`: "Error al actualizar la contraseña."
- ✅ No hardcoded text in component (all i18n'd)

---

## Files Verified

| File | Changes | Status |
|---|---|---|
| `src/web/handlers/user.rs` | change_me_password: validate + invalidate + return 200 | ✅ |
| `src/web/dto/user.rs` | ChangeSelfPasswordRequest with validators | ✅ |
| `src/modules/profile/pages/ProfilePage.vue` | Identity header + Seguridad section + changePassword logic | ✅ |
| `src/locales/es.json` | profile.security.* strings (12 entries) | ✅ |

---

## Defects

None detected. Password change feature is complete and correct.

---

## Edge Cases

**Session invalidation timing**
- Backend invalidates immediately after password change
- Frontend redirects after 1.5s
- User is logged out before redirect (session is invalid)
- Next login requires new auth

**Password mismatch feedback**
- Client-side error shown immediately (no API call)
- User sees green success message + toast, then 1.5s delay before logout
- Clear indication of what's happening

**i18n coverage**
- All user-facing text i18n'd
- No embedded strings
- Spanish localization complete
