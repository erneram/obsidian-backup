VERDICT: SHIP

## Hallazgos

- **Filtro Talonario**: `m.notes?.startsWith('Talonario')` correcto — optional chaining hace null-safe (null notes → no filtrado ✅). Riesgo residual: si el backend cambia el formato del note, los duplicados reaparecen. `startsWith('Talonario T-')` sería más preciso pero la granularidad actual cubre el caso documentado.

- **Migración 030**: `ADD COLUMN IF NOT EXISTS quantity NUMERIC(10,2), unit_price NUMERIC(10,2)` — idempotente, tipos consistentes con el schema existente ✅

- **PDF qty+precio**: El `collect` del PDF usa solo `(concept_label, description, amount)` — PACKAGE items muestran solo descripción+total, sin qty ni unit_price. Extras: `"Extra x{qty}"` en col1, `unit_price` silenciado (`let _ =`). Ponytail pre-existente al fix, Bug 4 diferido. Aceptable — la migración 030 habilita mejorarlo después.

- **Imprimir eliminado**: `Printer` ausente en imports (línea 606: `ArrowLeft, Plus, Minus, Trash2, Receipt, ChevronsUpDown, FileDown`). Sin `Imprimir` en template ✅. `Receipt` sigue importado — en uso en botón "Crear talonario" (línea 38) ✅
