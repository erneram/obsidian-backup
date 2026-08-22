# Fix Request — Editar qty SUPPLY ajusta stock físico

## Problema
Al editar un paquete aplicado en AdmissionDetailPage (dialog "Editar"):
- Eliminar un item SUPPLY: el stock físico en `inventory_items` no se restaura.
- Cambiar qty de un item SUPPLY (ej: de 5 a 3): la diferencia (2) no se devuelve al inventario.
- Cambiar qty al alza (ej: de 3 a 5): las 2 unidades adicionales no se descuentan.

## Root cause
`remove_item` (`DELETE FROM account_statement_items`) no toca `inventory_items`.
`submitEditPackage` hace delete+re-add pero el "re-add" usa `add_item` genérico que tampoco ajusta stock de SUPPLY.

## Fix esperado (PONYTAIL — mínimo diff)
En el backend, en `remove_item` / `remove_statement_item`: si el item tiene `inventory_item_id` (es SUPPLY), hacer:
```sql
UPDATE inventory_items SET quantity = quantity + <qty_eliminada>, last_updated = now()
WHERE id = inventory_item_id AND tenant_id = tenant_id
```
Y registrar un movement tipo 'IN' en `inventory_movements` (reversión).

Para cambio de qty: el frontend ya hace delete+re-add. Si `remove_item` restaura el stock al borrar, y `apply_package` descuenta al re-insertar con la nueva qty, el net result es correcto.

Verificar que `account_statement_items.quantity` se use como la cantidad a restaurar al borrar.
