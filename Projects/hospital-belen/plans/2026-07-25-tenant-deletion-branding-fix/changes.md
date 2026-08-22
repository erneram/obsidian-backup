# Changes — hospital-belen stage-2

## Files changed

### `hospital-belen-web/kairosaid/src/composables/useBranding.ts` (new)
Single exported `ref<TenantBranding | null>` singleton. No fetch — just reactive state shared between `main.ts` and `LoginPage.vue`.

### `hospital-belen-web/kairosaid/src/main.ts`
- Imports `activeBranding` from the new singleton.
- Sets `activeBranding.value = cached` on cache hit (sync, before first paint).
- Sets `activeBranding.value = branding` after network fetch resolves.
- Sets `activeBranding.value = null` in the null-fetch branch (tenant deleted) → LoginPage falls back to "Kairos Aid" instead of showing stale branding.

### `hospital-belen-web/kairosaid/src/modules/auth/pages/LoginPage.vue`
- Imports `activeBranding`.
- Shows `<img>` with `activeBranding.logoUrl` (if present) above the title.
- `<h2>` now reads `activeBranding?.name ?? 'Kairos Aid'` — falls back gracefully on apex/localhost.

### `hospital-belen-web/kairosaid/src/modules/platform/pages/PlatformRolesPage.vue`
**Bug fix (Task 2):** `moduleGroupState` now returns `selectedModuleIds.has(group.id)` when `mods.length === 0` (standalone group). Previously always returned `false` → checkbox never reflected selection and items appeared unsaved. `toggleModule` logic was already correct once the state function returns the right value — no change needed there.

**Permissions layout (Task 3):** Replaced flat `<label v-for>` list in permissions tabs with a 4-column matrix:
- Desktop: header row (Read · Create · Update · Delete) + one checkbox per action cell; missing action → `—`.
- Mobile (`sm:` breakpoint): stacks to labeled rows (action name + checkbox).
- `findPerm(perms, action)` helper added for the template lookup.
- `moduleAdminState` header checkbox and `togglePermsModule` / `togglePerm` behavior unchanged.

## What tester should review

1. **Login branding:** On a tenant subdomain (or `?tenant=<slug>` dev override), the login card must show the tenant logo (if set) and tenant name. On apex/localhost: "Kairos Aid", no logo.
2. **Modules round-trip:** Platform admin → tenant → role → Módulos → toggle a **standalone** module (e.g. Dashboard) AND a child module → Guardar → reopen → both stay checked. Was: standalone always appeared unchecked after save.
3. **Permissions matrix:** Permissions dialog now shows a 4-col grid per tab. Verify:
   - Each action cell has a checkbox if that action exists, `—` if not.
   - The admin header checkbox still selects/deselects all.
   - The `n/total` counter in the tab label stays correct.
   - Mobile: stacks to labeled rows.
4. Build: `npm run build` passes (verified, no errors).
