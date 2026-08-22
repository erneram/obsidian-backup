# Changes — hospital-belen — 4 issues batch

**Implementado:** 2026-06-26  
**Modo:** stage-2

---

## Fix (2026-06-26)

`dev_seed.rs` `ensure_test_warehouses`: quitado `created_by` del INSERT a `storages` (columna no existe en esa tabla) y corregido el unique constraint a `(warehouse_id, code)`.

---

## Archivos modificados / creados

| Archivo | Tipo | Descripción |
|---|---|---|
| `hospital-belen-api/migrations/028_extend_stmt_concept.sql` | Creado | Añade 10 valores en español al enum `stmt_concept` (idempotente con `IF NOT EXISTS`) |
| `hospital-belen-api/migrations/029_rename_default_tenant.sql` | Creado | Renombra slug del tenant `10000000-0000-0000-0000-000000000001` de `'default'` → `'hospital-belen'` |
| `hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionListPage.vue` | Modificado | `SelectTrigger class="w-full"` → `w-48` (línea 22) |
| `hospital-belen-api/src/infrastructure/dev_seed.rs` | Modificado | Añade `ensure_test_warehouses()` + llamada en `run()` tras `ensure_dev_appointments` |

---

## Qué revisar puntualmente (Tester)

### Issue 1 — migración 028
- Arrancar la API con una DB fresca: confirmar que la migración corre sin error.
- Crear un `account_statement_item` con concept `HOSPITALIZACION` y `OTROS` vía API — no debe devolver error de cast.
- `admission_extra_repository.rs:93` hardcodea `'OTROS'::stmt_concept` — verificar que el insert funciona post-migración.

### Issue 2 — migración 029
- Con DB fresca, arrancar la API: `ensure_demo_tenant("hospital-belen")` debe encontrar el tenant por slug y retornar `'10000000-0000-0000-0000-000000000001'` (no crear UUID nuevo).
- `GET /api/packages` como `admin@hospitalbelen.com` debe devolver los 116 paquetes de cirugía.

### Issue 3 — Vue w-48
- En pantalla ≥ 1024px: los filtros de AdmissionListPage deben mostrarse en una sola fila horizontal.
- El label más largo ("Parcialmente Pagado") debe caber dentro del SelectTrigger sin truncamiento visible.

### Issue 4 — dev_seed
- Arrancar la API: verificar en logs `Dev seed: warehouse TEST-A / storage GEN-A ensured` y `TEST-B / GEN-B`.
- `GET /api/warehouses` como admin debe incluir TEST-A y TEST-B.
- Cada storage debe tener 5 inventory_items (o menos si hay menos de 5 productos en catálogo).
- Segunda ejecución (restart): el seed debe ser idempotente — sin errores de conflicto.
