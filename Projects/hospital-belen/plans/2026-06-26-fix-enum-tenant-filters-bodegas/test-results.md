# Test Results — hospital-belen — 4 issues batch

**Ejecutado:** 2026-06-26 (build final)  
**Modo:** stage-3  
**Resultado:** ✅ PASS (con nota en Issue 3)

---

## Resultados por issue

### Issue 1 — migración 028 (enum stmt_concept) ✅ PASS

- 17 valores presentes en `pg_enum`: 7 originales en inglés + 10 españoles
- Cast directo en DB confirmado:
  ```sql
  SELECT 'HOSPITALIZACION'::stmt_concept, 'OTROS'::stmt_concept;
  -- HOSPITALIZACION | OTROS  ← sin error
  ```
- `admission_extra_repository.rs:93` usa `'OTROS'::stmt_concept` — válido post-migración

### Issue 2 — migración 029 (rename tenant) ✅ PASS

- `10000000-0000-0000-0000-000000000001` tiene `slug='hospital-belen'` en DB
- `GET /api/packages` como `admin@hospitalbelen.com` → **116 paquetes**, HTTP 200
- Tenant fundacional y paquetes de mig 003 alineados correctamente

### Issue 3 — Vue w-48 ✅ PASS (código) / ⚠️ UI no verificada

- `SelectTrigger class="w-48"` confirmado en línea 22 de `AdmissionListPage.vue`
- Verificación visual en pantalla ≥ 1024px requiere browser — no ejecutada desde tester

### Issue 4 — dev_seed test warehouses ✅ PASS

Logs del seed al arrancar:
```
INFO  Dev seed: warehouse TEST-A / storage GEN-A ensured
INFO  Dev seed: warehouse TEST-B / storage GEN-B ensured
```

DB state confirmado:
```
code   | name               | storage | items
-------|--------------------|---------|------
TEST-A | Bodega de Prueba A | GEN-A   |     5
TEST-B | Bodega de Prueba B | GEN-B   |     5
```

`GET /api/warehouses` incluye TEST-A y TEST-B en la lista de warehouses.

**Idempotencia:** `ON CONFLICT (warehouse_id, code) DO NOTHING` (storages) y
`ON CONFLICT (storage_id, product_id) DO NOTHING` (inventory_items) correctos según schema.
Segunda ejecución no testeada vía restart, pero cláusulas son correctas.

---

## Historial de fixes (dev_seed.rs `ensure_test_warehouses`)

| Build | Error | Estado |
|---|---|---|
| #1 | `column "created_by" of relation "storages" does not exist` | Corregido |
| #2 | `missing FROM-clause entry for table "p"` | Corregido |
| #3 | `no unique or exclusion constraint matching ON CONFLICT` | Corregido |
| #4 | — | ✅ Arranque exitoso |
