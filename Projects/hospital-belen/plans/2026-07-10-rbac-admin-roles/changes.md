# Changes — hospital-belen · stage-2

## Archivos modificados

### `hospital-belen-web/kairosaid/src/modules/admin/pages/RoleListPage.vue`
- **Permisos modal → matriz tabulada**: reemplaza lista plana por `Tabs/TabsList/TabsTrigger/TabsContent` agrupados por `resource`. Badge `n/total` en cada pestaña.
- **Grant rápido**: checkbox "Admin de este módulo" (tri-state: false/indeterminate/true) arriba de cada pestaña. Usa `moduleAdminState()` + `toggleModule()`.
- **Nuevo Rol — adminAllModules**: checkbox default `true`. Tras `createRole` exitoso, hace `PUT /roles/:id/permissions` con todos los IDs. Si el PUT falla → `toast.warning` (no silencioso).
- **allPermissions en `onMounted`**: ya no se carga solo al abrir el modal — se carga junto con `loadRoles()` al montar para estar disponible en `submitCreate`.
- **`permsByModule` computed**: agrupa `allPermissions` por `resource`, ordena acciones `read > create > update > delete > otros`.
- **Imports añadidos**: `Tabs, TabsList, TabsTrigger, TabsContent` de `@inksightdev/ui`; `useI18n` de `vue-i18n`; `computed`.

### `hospital-belen-web/kairosaid/src/locales/es.json`
- Añade bloque `admin.roles.*`: labels de módulos (`patients`→"Pacientes", etc.), copy de checkboxes y toasts.

## Sin cambios
- Backend / migraciones — RBAC ya soporta todo.
- `adminService.ts` — sin helpers nuevos, el page llama `apiService` directo.
- `PermissionMatrix.vue` — no creado (archivo final ~480 líneas, cerca del umbral, inline suficiente).

## Qué revisar puntualmente (Tester)

1. **Modal permisos**: pestañas renderizan correctamente, badge actualiza en tiempo real al marcar/desmarcar.
2. **Admin de este módulo**: tri-state funciona (vacío → indeterminate → lleno y viceversa).
3. **Nuevo Rol + adminAllModules=on**: el rol se crea Y los permisos se asignan en el PUT. Verificar que el rol quede con todos los permisos.
4. **Nuevo Rol + adminAllModules=off**: solo se crea el rol, sin PUT de permisos.
5. **PUT permisos falla**: aparece `toast.warning`, no `toast.error` silencioso.
6. **Superadmin**: botón Permisos visible pero permisos deshabilitados (comportamiento existente, no debe romperse).
7. **allPermissions vacío**: mensaje "No hay permisos creados." en modal, sin pestañas.
8. **i18n labels**: módulos desconocidos hacen fallback a capitalize(resource), no arrojan error.
