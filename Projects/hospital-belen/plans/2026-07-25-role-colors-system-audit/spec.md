# Spec — Role colors system + profile fix + audit (hospital-belen)

Stage-1 · effort=high · repos clean on `main`. Supersedes prior spec (login/modules/permissions items folded into Audit §1 as still-open).

Pages: platform roles `…/platform/roles` · users `…/platform/users` · profile `hospital-belen.…/profile`

---

## 1. Code audit — real defects found (fix or flag)

Substantiated during exploration, ranked:

1. **`profile/services/profileService.ts` is 100% fake** — `DUMMY_PROFILE` ("María Fernanda López"), hardcoded `ROLE_PERMISSIONS`, `USER_OVERRIDES`, `setTimeout` fake latency. This IS the "hardcoded names/roles/members since". Fixed in §4.
2. **`services/catalog.service.ts` — dummy data**, `// TODO: Uncomment when backend is ready`. Blood types/allergies never hit the API. Flag; out of scope unless requested.
3. **`stores/appointmentsStore.ts`** — `fetchAppointmentsToday`/`updateAppointmentStatus` are `setTimeout` simulations, no API. Flag.
4. **`services/growthChart.service.ts`** — suspected mock (same family). Coder: verify, flag only.
5. **Modules dialog standalone bug (still open from prior spec):** `PlatformRolesPage.vue` `moduleGroupState`/`toggleModule` ignore standalone groups (own id, no children) → never render checked / don't persist. Fix if not already done.
6. `console.*` (4) and `as any`/`: any` (14) — note count, don't chase.

Audit output: coder appends a short `## Audit` list to `.pipeline/changes.md` (file:line + one-line problem) for items 2–4/6 not fixed, so they're tracked not lost.

---

## 2. Roles list UX — superadmin first + color

**File:** `hospital-belen-web/kairosaid/src/modules/platform/pages/PlatformRolesPage.vue`

- **Sort:** after `loadRoles`, order `is_superadmin` first, then by `name`. Client-side sort on `roles.value` (backend `list` orders by name only; don't touch SQL). Keep pagination — sort within the loaded page.
- **Color differentiation:** render a color dot/left-border per row using the new `role.color` (§3). Superadmin keeps its existing purple pill (lines 54–59); regular roles show their assigned color via the shared helper (§5). Null color → neutral `bg-muted` swatch.

---

## 3. Role colors — schema + backend + admin form

### 3a. Migration (mirror `007_tenant_branding.sql` exactly)
New `hospital-belen-api/migrations/008_role_color.sql`:
```sql
ALTER TABLE roles ADD COLUMN color varchar(7);
ALTER TABLE roles ADD CONSTRAINT roles_color_hex CHECK (color ~ '^#[0-9A-Fa-f]{6}$');
```
Nullable → NULL = neutral/derived badge; no backfill (every existing role keeps working).

### 3b. Backend plumbing (color must flow through every roles read/write)
- `idens.rs` `RoleIden`: add `Color`.
- `domain/rbac/role.rs`: add `pub color: Option<String>` to `Role`, `RoleForCreate`, `RoleForUpdate`.
- `role_repository.rs`: add `RoleIden::Color` to the SELECT column lists in `get` (line ~64) and `list` (~97), and to the INSERT (`create`, ~179) and UPDATE (`update`, ~122) builders. **Explicit sea-query columns — the new column will NOT appear unless added by hand.**
- `web/dto/rbac.rs`: add `color: Option<String>` to `CreateRoleRequest` and `UpdateRoleRequest`.
- `web/handlers/platform_rbac.rs`: pass `color: body.color` in `create_role_in_tenant` (`RoleForCreate`) and `update_role_in_tenant` (`RoleForUpdate`).
- Validation: hex `#RRGGBB` server-side is the CHECK constraint; also validate in `CreateRoleRequest` (existing `ValidatedJson` path) to return 400 not 500. `update_role_in_tenant` uses plain `Json` — add a light guard or move to `ValidatedJson`.
- Result: `GET /api/roles` (tenant) and `GET /api/platform/tenants/{tid}/roles` both return `color`. Confirm `rbac_handlers` tenant create/update role also carry color (same DTOs at `/api/roles`).

### 3c. Admin form (color picker)
`PlatformRolesPage.vue` create + edit dialogs: add a color field. Use native `<input type="color">` (native platform feature — no picker lib) bound to `createForm.color` / `editForm.color`, with a "sin color" clear option (null). Add `color` to `admin.ts` `Role` type + create/update service payloads (`platformAdmin.service.ts`).

---

## 4. Profile page — real data + role color

**Files:** `profile/services/profileService.ts`, `profile/composables/useProfile.ts`, `profile/pages/ProfilePage.vue`, `profile/types/index.ts`

Backend already has `GET /api/users/me` (`user.rs::get_me`) → `UserResponse { first_name, last_name, email, roles: string[] (slugs) … }`. Two gaps:

- **`UserResponse` has no `created_at`** → "member since" has no source. Add `created_at` to `UserResponse` (`web/dto/user.rs`) + its `From<User>` impl (users table has `created_at`). `get_me` then returns it.
- **Role color for profile:** profile runs as a tenant user. Fetch `GET /api/roles` (returns color per §3b), map `slug → { name, color }`, resolve against `me.roles[0]` (primary role).

Frontend:
- **Delete** `DUMMY_PROFILE`, `setTimeout`, and rebuild `profileService` on `apiService`: `getProfileSummary()` → `GET /users/me` mapping `name = first_name + ' ' + last_name`, `memberSince = created_at`, `role = roles[0]`.
- `ProfilePage.vue`: `displayName`, `roleLabel`, `memberSinceLabel` already computed — they'll now bind real data. Render role name as a colored badge via the shared helper (§5), not the raw i18n key.
- **OPEN QUESTION:** the permission buckets (`ROLE_PERMISSIONS`/`USER_OVERRIDES` dummies feeding `roleAllowed/effective…`) are also fake. The request names only name/role/member-since + color. Wiring buckets to real effective-permissions needs a backend endpoint that doesn't obviously exist yet — **out of scope unless confirmed**. If in scope, name the endpoint and I'll extend the spec.

---

## 5. Cross-page color consistency — one shared helper

Create `src/utils/roleColor.ts` — single source so roles list, user-assignment, and profile render identically:
```ts
export function roleColor(role: { slug: string; is_superadmin?: boolean; color?: string | null }): string
// superadmin → fixed purple; role.color if set; else deterministic fallback from slug hash → fixed palette. Returns a hex.
```
Provide a companion `roleBadgeClass`/inline-style so a `color` drives text/bg with adequate contrast (tint bg + solid text). Consumers:
- **Roles list** (§2) — row swatch.
- **User assignment** — `platform/components/TenantAccordion.vue` roles dialog (`role.name`, line ~149) + the assigned-role display: show the color dot/badge. Add `color` to its `Role` import usage.
- **Profile** (§4) — role badge.

`// ponytail:` the slug-hash fallback (fixed ceiling: collisions possible across many roles; upgrade path = require color on create).

## Diseño visual

Admin/utility surface — reuse the existing design system, no new palette. The role color IS the accent; keep chrome quiet.
- **Badge:** rounded-full, tinted background at ~15% + solid role color text (mirror the existing superadmin purple pill treatment, lines 54–59), so custom colors sit in the same visual family. Contrast floor: solid text on tint, never text-on-saturated.
- **Color picker:** native `<input type="color">` + a small clear ("sin color") link; label "Color del rol". No third-party picker.
- **Roles list:** 3px left color border on the row OR a leading dot — pick the dot (less layout risk in the existing table). Superadmin row unchanged.
- Dark mode: tint via `/30` opacity like existing `dark:bg-purple-900/30` patterns so colors survive both themes.

---

## Verification (tester)
- Backend: `cd hospital-belen-api && cargo test && cargo clippy`; run migration 008 against a scratch DB, confirm CHECK rejects `#zzz`/`red` and accepts `#0F766E`.
- Frontend: `cd hospital-belen-web/kairosaid && npm run build && npm run lint`.
- Behavioral:
  - Create a role with a color → appears in roles list swatch, in user-assignment dialog, and on a user's profile with that color.
  - Superadmin sorts first and keeps purple.
  - Profile shows the logged-in user's real name/role/member-since (no "María Fernanda López").
  - Round-trip color edit + null-clear.

## Existing patterns to follow
- Migration + hex CHECK: `migrations/007_tenant_branding.sql`.
- Role read/write plumbing: `role_repository.rs` explicit `RoleIden` column lists (add `Color` everywhere they list columns).
- Colored pill: existing superadmin span in `PlatformRolesPage.vue` (54–59).
- `me` endpoint shape: `user.rs::get_me` + `dto/user.rs::UserResponse`.

No installable skill applies (`npx skills find "role color rbac badge"` → nothing beats in-repo patterns).
