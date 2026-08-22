# Changes — admissions: fix 500 en items, Extras invisibles, date filters UI

## Archivos modificados

### Backend
**`hospital-belen-api/src/infrastructure/database/repositories/admission_repository.rs`**
- `add_item` (~L377): RETURNING ahora incluye `NULL::text AS storage_name`
- `apply_package` (~L519): RETURNING ahora incluye `NULL::text AS storage_name`
- Ambos sitios carecían del campo → `ColumnNotFound` de sqlx → 500
- GET (~L192) ya tenía `s.name AS storage_name` vía JOIN — no tocado

### Frontend
**`hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionDetailPage.vue`**
- `conceptualItems` (L907): removida exclusión de `'OTROS'`
- Antes: `i.concept !== 'PACKAGE' && i.concept !== 'OTROS' && i.concept !== 'OTHER'`
- Después: `i.concept !== 'PACKAGE' && i.concept !== 'OTHER'`
- `OTHER` (inglés) sigue excluido → va a tabla Cargos de Inventario

**`hospital-belen-web/kairosaid/src/modules/admissions/pages/AdmissionListPage.vue`**
- Import `DatePicker` agregado desde `@inksightdev/ui`
- `dateFrom`/`dateTo`: `ref('')` → `ref<Date>()`
- `toYmd(d?: Date)` helper: formatea fecha local a `YYYY-MM-DD` sin timezone shift
- `load()`: usa `toYmd(dateFrom.value)` y `toYmd(dateTo.value)`
- Markup: `<Input type="date">` reemplazados por `<DatePicker locale="es-GT" @update:modelValue="load(1)">`
- `clearFilters()`: asigna `undefined` en vez de `''`

## Verificación grep AccountStatementItem

Resultado de `grep -rn "query_as::<_, AccountStatementItem>" src/`:
- L192 (`get`) — ya tenía `storage_name` vía JOIN ✓
- L371 (`add_item`) — **fixed** ✓
- L512 (`apply_package`) — **fixed** ✓
- `replace_inventory_items` (~L863) — no usa `query_as::<_, AccountStatementItem>` ✓

## Qué debe revisar el Tester

1. **Backend 500 fix:** `POST /api/admissions/{id}/items` body `{"concept":"MEDICAMENTOS","description":"x","amount":10}` → debe retornar 201 con item JSON (campo `storageName: null`), no 500.
2. **apply_package:** `POST /api/admissions/{id}/packages` → sigue retornando 201. No romper.
3. **OTROS visible:** Agregar Extra con concepto `OTROS` → aparece en tabla Extras (no desaparece).
4. **MEDICAMENTOS visible:** Extra con concepto `MEDICAMENTOS` → aparece en tabla Extras.
5. **Cargos de Inventario:** items con concepto `OTHER` siguen en tabla Cargos de Inventario.
6. **DatePicker:** filtros "Desde"/"Hasta" usan componente del design system. Params enviados al backend en formato `YYYY-MM-DD`. Limpiar filters → campos vacíos, sin params en request.
7. **cargo check / vue-tsc:** ya validados — ambos EXIT 0.
