VERDICT: SHIP

## Razonamiento
- **Gating correcto y verificado**: `AppRole` niveles PlatformAdmin(5) > TenantAdmin(4). `mw_require_min_role` usa `>=` sobre `level()`. rbac_platform gate=PlatformAdmin → tenant_admin(4) recibe 403; rbac_tenant gate=TenantAdmin → tenant_admin pasa. Correcto.
- **Decoupling is_superadmin ✅**: `ctx.role()` se deriva del `token.role` (JWT), NO de `is_superadmin`. tenant_admin con is_superadmin=true resuelve a `AppRole::TenantAdmin` → bloqueado en gate platform. `get_modules` filtra por `is_platform_admin()` (nivel de rol), no por el flag. Exactamente lo que pedía el spec.
- **Orden de middleware**: `.route_layer(min_role).route_layer(mw_ctx_require)` — idéntico al grupo `tenants` (precedente probado). ctx (auth) es outermost → corre primero, luego role check. Correcto.
- **Migración 031**: idempotente (DELETE por join slug+name), quirúrgica, no toca is_superadmin ni role_permissions. Comentario ponytail presente.
- **Sidebar sin dead-links**: section-admin sólo tiene 4 hijos (admin-tenants, users-list, roles-list, endpoints); 031 quita 3, queda users-list → sección renderiza. Las rutas platform sin menu_item (permissions, endpoint-groups, menu-items, audit-logs) no dejan links huérfanos. Verificado contra 002_seed_rbac.sql.
- Sin push/commit/PR ni Co-Authored-By. Cambios dentro de scope.

## Hallazgos
🟡 MEDIO — **audit-logs desvía del spec**: spec OPEN QUESTION 2 resolvió "audit-logs visible para tenant_admin = SÍ" y lo puso en el grupo tenant-admin. El código lo movió a rbac_platform (PlatformAdmin → tenant_admin recibe 403), comentado "platform-only per DISPATCH" — pero el DISPATCH no mencionó audit-logs. Es más restrictivo (dirección segura) y no hay menu_item de auditoría, así que impacto práctico bajo, pero es una decisión escrita del spec revertida sin flag. Confirmar con humano cuál es la intención.
🟡 MEDIO — **drop de grupos vacíos es efectivamente no-op** (module.rs:85-91): `group_type` sólo pasa a "group" dentro del loop que empuja un hijo (l.85), que a la vez llena `modules`. Por tanto nunca existe un grupo con `group_type=="group"` y `modules` vacío → `retain(|g| g.group_type=="standalone" || !modules.is_empty())` no descarta nada. Una sección padre que pierda TODOS sus hijos conserva group_type="standalone" y sobrevive como header vacío. No se dispara con los seeds actuales (toda sección conserva ≥1 hijo), pero el Point 6 que el tester marcó PASS no hace lo que dice. Fix: marcar parents como "group" al crearlos (por parent_id NULL + path NULL) y dropear los vacíos.
🟢 BAJO — `get_modules` corre 2 queries extra (get_user_roles + get_items_for_roles) por request para no-platform-admins; aceptable, pero es el hot path del sidebar. Cachear si se nota.

## Nota
Tests fueron análisis estático (sin DB/servidor). El gating core — lo que pide el DISPATCH — está bien y lo tracé contra el código real. Los dos MEDIO no bloquean pero conviene resolver el de audit-logs (decisión de producto) antes de merge y anotar el retain como deuda.
