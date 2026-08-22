# Spec — Profile page audit + enhancement (hospital-belen)

Stage-1 · effort=medium · Prior role-color/profile work is **implemented but uncommitted** in `hospital-belen-web` (10 modified files). This dispatch audits that work and polishes the profile.

Page: `hospital-belen.kairosaid.inksightdev.com/profile`
Files: `src/modules/profile/{pages/ProfilePage.vue, composables/useProfile.ts, services/profileService.ts, types/index.ts, components/PermissionBuckets.vue}`, `src/locales/es.json`, plus API for password (below).

---

## 1. Hardcoded-data audit — result: mostly clean, 2 gaps

Identity/roles/member-since are now wired to real API (`/users/me`, `/users/me/roles`, `/users/me/permissions`) via `profileService`. Remaining hardcoded values:

- **Change-password section is not i18n'd** — heading `"Cambiar Contraseña"`, the two `<label>`s, placeholder `"Mínimo 8 caracteres"`, and every `pwError`/`pwSuccess` string are inline Spanish while the rest of the page uses `t()`. Add a `profile.password.*` block to `es.json` and replace the literals.
- **Dead i18n keys** — `profile.roles.*`, `permissionsByRole*`, `permissionsByUser*` are no longer referenced (role label now comes from `profile.roleName`; only the effective bucket renders). Remove them (or flag). Low priority.

No fake names remain (`defaultName` = "Usuario").

---

## 2. Change password E2E — BROKEN for normal users (primary finding)

**File:** `ProfilePage.vue` `changePassword()` → `PUT /users/{uid}/password`.

Backend (`user.rs::change_password`, `router.rs:146`):
```
PUT /api/users/{id}/password  → route_layer mw_require_permission(AppPermission::UserUpdate)
ChangePasswordRequest { new_password }   // no current-password field
```

Two real defects:

1. 🔴 **Wrong endpoint for self-service.** This route requires the `user:update` permission. A normal tenant user (doctor, receptionist…) hitting their own profile password form gets **403** — the feature silently fails for exactly the people it's for. Conversely a user *with* `user:update` can reset **any** user's password with **no current-password check**.
   → **Fix (backend):** add `PUT /api/users/me/password` — authenticated, **self only**, body `{ current_password, new_password }`, verifies `current_password` before setting (reuse `UserService::change_password` after a verify step). No special permission gate. Profile calls this instead of `/users/{id}/password`. Keep the admin `/users/{id}/password` route as-is for platform admin.

2. 🔴 **Session is invalidated on success** (`change_password` calls `invalidate_session` + `invalidate_user_permissions`). The user's token is dead immediately, but the UI shows "Contraseña actualizada correctamente" and the next API call 401s with no explanation.
   → **Fix (frontend):** on success, message the user that they must sign in again and redirect to login (call `authStore.logout()` then `router.push('/login')` after a short toast). Don't leave them in a zombie session.

Secondary:
- 🟡 Add a **current password** input (required by the new endpoint).
- 🟡 `pwSuccess`/`pwError` never reset except on next submit — clear them when the user edits a field.
- 🟡 Keep the 8-char min (matches backend `#[validate(length(min = 8))]`); surface the backend's validation error text on 400.

**Manual E2E (tester):** log in as a NON-admin tenant user → change password with correct current password → expect success + forced re-login → new password works, old fails. Wrong current password → clear error, no change.

---

## 3. UI implementation review — primitive inconsistency

The page mixes design-system components with raw Tailwind, unlike sibling pages (`PlatformRolesPage.vue` uses bare `<Input class="mt-1" />`, `<Label>`, bare `<Button>`):

- Password `<Input>`s carry `class="w-full border border-input rounded-md px-3 py-2 text-sm bg-background"` — the `@inksightdev/ui` `Input` already styles itself; strip the redundant classes.
- Raw `<label class="text-sm font-medium">` → use the `Label` component.
- `<Button>` carries raw color/padding classes → use the bare `<Button>` variant like elsewhere.
- Role badge inline-style via `roleBadgeStyle` is fine; keep it.

Standardize on the design-system primitives so the profile matches every other form in the app.

---

## 4. UI enhancement

## Diseño visual

Self-service profile inside an established design system — the job is a **calm, legible identity + a clear security section**, not a showpiece. Reuse `@inksightdev/ui` + semantic tokens; the one accent is the role color. No new palette, no motion beyond default.

**Layout (two clear zones):**
```
┌───────────────────────────────────────────┐
│  ●avatar   Nombre Apellido                 │  ← identity header
│            [rol badge·color]  ·  email     │
│            Miembro desde 14 mar 2022       │
├───────────────────────────────────────────┤
│  Permisos efectivos            (bucket)    │
├───────────────────────────────────────────┤
│  🔒 Seguridad — Cambiar contraseña          │  ← distinct section, section heading
│  Contraseña actual                          │
│  Nueva contraseña   Confirmar               │
│                       [Actualizar]          │
└───────────────────────────────────────────┘
```

- **Identity header:** promote email into the header line (currently absent from the card), inline with the role badge; member-since gets a small `lucide` calendar icon to read as metadata, not body text. Avatar stays the initials/User circle — if `first_name`/`last_name` present, show initials instead of the generic icon (one deliberate touch that personalizes without new assets).
- **Type:** keep the existing scale (`text-3xl` page title, `text-xl font-semibold` name). Role badge in the existing tinted-pill treatment (`roleBadgeStyle`). Don't introduce new display faces — this is a utility page.
- **Security section:** give it its own `Card` with a labelled heading ("Seguridad") and a lock icon so it reads as a separate concern from identity; consistent `Label`/`Input`/`Button` (§3); inline validation states in the destructive/success tokens already used.
- **Empty/error states:** the existing `loadError`/`loading` cards are fine — keep. Password success → toast + redirect (§2), not a permanent green line.
- Responsive: header stacks on mobile (already `sm:flex-row`); password grid already `sm:grid-cols-2`. Verify focus rings visible on all inputs/buttons (accessibility floor).

Restraint: the signature is the **initials avatar + role-color badge**; everything else stays quiet.

---

## Verification (tester)
- Frontend: `cd hospital-belen-web/kairosaid && npm run build && npm run lint`.
- Backend (if `/users/me/password` added): `cd hospital-belen-api && cargo test && cargo clippy`.
- Behavioral: §2 password E2E as a non-admin user; profile renders real name/role/email/member-since; role badge color matches roles list + user-assignment (cross-page consistency from prior spec).

## Existing patterns to follow
- Form primitives: `PlatformRolesPage.vue` create/edit dialogs (`Label` + bare `Input`/`Button`).
- Self endpoints: `user.rs::get_me`, `get_me_roles`, `get_me_permissions` (`router.rs:121-123`) — add `me/password` beside them.
- Role badge: `src/utils/roleColor.ts` (`roleColor`, `roleBadgeStyle`) — already used, reuse.
- Password verify: `UserService::change_password` — wrap with a current-password check for the self route.

OPEN QUESTION: none blocking. Confirm whether the admin `/users/{id}/password` route should additionally start requiring current-password (out of scope here — this spec only fixes the self-service path).
