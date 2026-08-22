VERDICT: SHIP

## Razonamiento
- Los 3 fixes coinciden exactamente con el spec, sin scope creep.
- Verificación independiente (no solo tests verdes): leí el diff real en ambos submódulos + contrato d.ts de DatePicker.
- Sin triggers de BLOCK: no push, no `Co-Authored-By`, nada fuera de scope.

## Hallazgos

### Issue 1 — backend 500 (add_item / apply_package)
🟢 Correcto. Ambos RETURNING agregan `NULL::text AS storage_name` con cast explícito. Todos los `query_as::<_, AccountStatementItem>` (get vía JOIN, add_item, apply_package) devuelven la columna. `replace_inventory_items` no usa ese struct → sin bug latente. FromRow matchea por nombre, orden irrelevante.

### Issue 2 — Extras invisibles
🟢 Correcto. `conceptualItems` (L907) ahora solo excluye PACKAGE/OTHER → OTROS visible en tabla Extras. `inventoryChargeItems` sigue `=== 'OTHER'` (Cargos de Inventario). Distinción OTHER(ing)/OTROS(esp) respetada. `extras` (L748) es un ref de endpoint distinto (enfermería), no afectado — sin regresión.

### Issue 3 — DatePicker
🟢 Correcto y type-safe. `modelValue?: Date` en d.ts ↔ `ref<Date>()` (Date|undefined). Emit `update:modelValue:(date:Date)` ↔ `@update:modelValue="load(1)"`. Props `placeholder`/`locale` existen. `toYmd` usa fecha local (evita corrimiento por timezone) → `undefined` cuando vacío, no `''`. `clearFilters` a `undefined`. `Input` sigue importado (patientSearch L15) → sin import muerto.

### Tests
🟢 `tests/admission_items.rs` sigue el patrón del repo (`common::login`/`api_url`), consistente con auth_login/permissions.

## Notas no bloqueantes
🟡 Tests de integración requieren API corriendo (localhost) → no corren en CI sin server. Documentado por tester, aceptable.
🟢 Archivos sueltos fuera del cambio: `.DS_Store` (api) y `test-toast-css.mjs` (web) untracked — no incluir en el commit.
🟢 Migración 028 (`ADD VALUE`) requiere PG12+ en prod para que existan HOSPITALIZACION/MEDICAMENTOS/OTROS — verificar antes de cerrar (flag del spec).
